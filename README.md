# class_GEE
Clasificacion Google Earth Engine

// Delimitacion del area de estudio

Map.addLayer(lim, {color:'#ff1cdb'}, 'Límite', true, 0.5);
Map.centerObject(lim,10);

// Seleccion de imagenes 

  // Seleccion de coleccion
  //Función enmascarrar nube sentinel
function maskS2clouds(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000)
   .copyProperties(image, ["system:time_start"]);
}

//Función NDVI
var ndvi = function(image){
  var ndvi = image.normalizedDifference(["B8","B4"]).rename("NDVI");
  return image.addBands(ndvi);
};

// Predictores 
 var addind = function(image) {
     var ndvi = image.normalizedDifference(['B8','B4']).rename('NDVI').double()// Normalized Difference Vegetation Index
     var ndwi = image.normalizedDifference(['B3','B8']).rename('NDWI').double()// Normalized Difference Water Index
     var ndbi = image.normalizedDifference(['B12','B8']).rename('NDBI').double()// Normalized Difference Built Index
     return ndvi.addBands(ndwi).addBands(ndbi);
   };


//Funcion de corte
function corte(image){
  return image.clip(lim); //es necesario declarar la capa de corte antes e incluirla en la funcion 
}

// Map the function over one year of data and take the median.
// Load Sentinel-2 TOA reflectance data.
//Filtramos nuestra imagen
var dataset = ee.ImageCollection('COPERNICUS/S2_SR')
                  .filterDate('2020-01-01', '2020-10-31')//Filtro de fechas
                  // Pre-filter to get less cloudy granules.
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 5))//Filtro por metadata
                  .filterBounds(lim) //Filtro por zona
                  //.map(ndvi) //Agregamos NDVI como una banda
                  .map(corte)
                  .map(maskS2clouds);
print(dataset);
  
////  Aplica función addind sobre coleccion cloudMaskedSR 
  var ind= dataset.map(addind); 
  print(ind, 'Índices');
  
////  Aplicacion de reductor a  índices
  var ind_reductor = ind.max().clip(lim);
  print(ind_reductor, 'Mediana Indices');
Map.addLayer(ind_reductor, {bands: ['NDVI'], min:-0.5, max: 1}, 'NDVI_reductor');
  
//// Cálculo mediana colección Sentinel 2 
  var Sentinel_mediana = dataset.median()// calcula mediana de cada pixel de la colección 
    .clip(lim); // corta por objeto lim
    
//// Variables topográficas 
  var dem = ee.Image('USGS/SRTMGL1_003');
  var topo = ee.Terrain.products(dem).clip(lim);
print (topo, 'variables topograficas')
  
Map.addLayer(topo, {bands: ['slope']}, 'pendiente');

/*
////  Unir bandas S2 + índices + topográficas
  var bandas = ee.Image.cat([Sentinel_mediana,ind_reductor,topo]);
  var bandas_ =['B2','B3','B4','B5','B6','B7','B8','B11','B12', 'NDVI','NDWI','NDBI','elevation','slope','aspect'];
  //print(bandas, 'bandas');
  //print(bandas_,'bandas_')
*/

// Load the Sentinel-1 ImageCollection.
var sentinel1 = ee.ImageCollection('COPERNICUS/S1_GRD');

//Sentinel
var vh = sentinel1
  // Filter to get images with VV and VH dual polarization.
  .filterDate('2020-01-01', '2020-10-31')//Filtro de fechas
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
  // Filter to get images collected in interferometric wide swath mode.
  .filter(ee.Filter.eq('instrumentMode', 'IW'));

// Filter to get images from different look angles.
var vhAscending = vh.filter(ee.Filter.eq('orbitProperties_pass', 'ASCENDING'));
var vhDescending = vh.filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'));

// Create a composite from means at different polarizations and look angles.
var bandas_radar = ee.Image.cat([
  vhAscending.select('VH').mean(),
  ee.ImageCollection(vhAscending.select('VV').merge(vhDescending.select('VV'))).mean(),
  vhDescending.select('VH').mean()
]).focal_median();

//var bandas_radar =['VH','VV','VH_'];

// Display as a composite of polarization and backscattering characteristics.
//Map.addLayer(bandas_radar, {min: [-25, -20, -25], max: [0, 10, 0]}, 'bandas_radar');

/*
/////classific radar

//// Extraer información desde las bandas a partir de los puntos 
    var Puntos_clas1 = bandas.select(bandas_radar).sampleRegions({
      collection: ptos_todos, 
      properties: ['class'],
      scale: 30
      });
      
  print(Puntos_clas1, 'Puntos clas1');//// Preparación puntos
  
    ////  Partición de los datos: 70% entrenamiento y 30% control  
    
  var trainingTesting1 = Puntos_clas1.randomColumn();// crea una columna con n° pseudo-aleatorios entre 0 y 1 
  var trainingSet1 = trainingTesting1
     .filter(ee.Filter.lessThan('random', 0.7));// seleccionada de la columna 'random' valores <= a 0.7
  var testingSet1 = trainingTesting1
     .filter(ee.Filter.greaterThanOrEquals('random', 0.7));// seleccionada de la columna 'random' valores >= a 0.7   
   
   
////  Visualizacion clasificacion /Creación paleta de colores
  var paleta = [
     '#1c45ff', // Agua
     '#dd52da', // Suelo
     '#4e6f5c', // Urbano
     '#490186', // Humedal
     '#10aa3d',// Vegetacion
     '#ebf2e5' // Nieve
     ];  
     

//// Clasificación con Random Forest 
  var clasificadorRF1 = ee.Classifier.smileRandomForest(50).train({
   features: trainingSet1, // usando puntos de entrenamiento
   classProperty: 'class', // clase 
   inputProperties: bandas_radar // bandas usadas 
   });

  var clasificadaRF1 = bandas.select(bandas_radar).classify(clasificadorRF1);// Clasifica imagen*/
/*
 Map.addLayer(clasificadaRF1, {min: 0, max: 5, palette:paleta }, 'clasificación Random Forest_radar'); 
  
  
 //// Validacion
 
 var confusionMatrix1 = ee.ConfusionMatrix(testingSet1.classify(clasificadorRF1)
   .errorMatrix({
    actual: 'class', 
    predicted: 'classification'
    }));

 var pg1 = confusionMatrix1.accuracy();// exactitud general
 var pu1 = confusionMatrix1.consumersAccuracy();// exactitud usuario
 var pp1 = confusionMatrix1.producersAccuracy();// extactitud productor
 var k1 = confusionMatrix1.kappa(); // kappa
 
 print('Confusion matrix ', confusionMatrix1);
 print('Overall Accuracy ', pg1);
 print('Producers Accuracy ', pp1);
 print('Consumers Accuracy ', pu1);
 print('Kappa ', k1);
*/

////  Unir bandas S2 + índices + topográficas + Radar
  var bandas = ee.Image.cat([Sentinel_mediana,ind_reductor,topo, bandas_radar]);
  var bandas_ =['B2','B3','B4','B5','B6','B7','B8','B11','B12', 'NDVI','NDWI','NDBI','elevation','slope','aspect','VH','VV'];
  //print(bandas, 'bandas');
  print(bandas_,'bandas_');

Map.addLayer(Sentinel_mediana,{bands:'B2,B8,B4'},'sentinel_mediana'); 
  
//// Preparación puntos
  
    //// Unir puntos/polígonos en un objeto
    var ptos_todos = Agua.merge(Suelo).merge(Urbano).merge(Humedal).merge(Vegetacion).merge(Nieve);
  print(ptos_todos, 'Puntos todos');
  
 //// Extraer información desde las bandas a partir de los puntos 
    var Puntos_clas = bandas.select(bandas_).sampleRegions({
      collection: ptos_todos, 
      properties: ['class'],
      scale: 30
      });
  print(Puntos_clas, 'Puntos clas');//// Preparación puntos
  
    ////  Partición de los datos: 70% entrenamiento y 30% control  
    
  var trainingTesting = Puntos_clas.randomColumn();// crea una columna con n° pseudo-aleatorios entre 0 y 1 
  var trainingSet = trainingTesting
     .filter(ee.Filter.lessThan('random', 0.7));// seleccionada de la columna 'random' valores <= a 0.7
  var testingSet = trainingTesting
     .filter(ee.Filter.greaterThanOrEquals('random', 0.7));// seleccionada de la columna 'random' valores >= a 0.7   
   
  //print(trainingSet, 'training set');
  //print(testingSet, 'testing set'); 
  
 //// Clasificación con Mínima Distancia
  var clasificadorMD = ee.Classifier.minimumDistance().train({
   features: trainingSet, // usando puntos de entrenamiento
   classProperty: 'class', // clase 
   inputProperties: bandas_ // bandas usadas 
   });
   
  var clasificadaMD = bandas.select(bandas_).classify(clasificadorMD);// Clasifica imagen
   
////  Visualizacion clasificacion /Creación paleta de colores
  var paleta = [
     '#1c45ff', // Agua
     '#dd52da', // Suelo
     '#4e6f5c', // Urbano
     '#490186', // Humedal
     '#10aa3d',// Vegetacion
     '#ebf2e5' // Nieve
     ];  
     
Map.addLayer(clasificadaMD, {min: 0, max: 5, palette:paleta }, 'clasificación Minima Distancia');

//// Clasificación con Random Forest 
  var clasificadorRF = ee.Classifier.smileRandomForest(50).train({
   features: trainingSet, // usando puntos de entrenamiento
   classProperty: 'class', // clase 
   inputProperties: bandas_ // bandas usadas 
   });

  var clasificadaRF = bandas.select(bandas_).classify(clasificadorRF);// Clasifica imagen*/

 Map.addLayer(clasificadaRF, {min: 0, max: 5, palette:paleta }, 'clasificación Random Forest'); 
  
  
 //// Validacion
 
 var confusionMatrix = ee.ConfusionMatrix(testingSet.classify(clasificadorRF)
   .errorMatrix({
    actual: 'class', 
    predicted: 'classification'
    }));

 var pg = confusionMatrix.accuracy();// exactitud general
 var pu = confusionMatrix.consumersAccuracy();// exactitud usuario
 var pp = confusionMatrix.producersAccuracy();// extactitud productor
 var k = confusionMatrix.kappa(); // kappa
 
 print('Confusion matrix ', confusionMatrix);
 print('Overall Accuracy ', pg);
 print('Producers Accuracy ', pp);
 print('Consumers Accuracy ', pu);
 print('Kappa ', k);
 
 
/// Exporta imagen Landsat (mediana) a Google Drive.
   Export.image.toDrive({
    image: bandas_,
    description: 'Exportar_landsat',
    fileNamePrefix: 'Landsat_mediana',
    folder: 'Diplomado_GEE',
    scale: 30,
    region: lim,
    crs:'EPSG:32718'
  });
 
///  Exporta clasificación como Asset 
   Export.image.toAsset({
    image: clasificadaMD,
    description: 'Clasificada_Asset',
    assetId: 'clasificacion',
    scale: 30,
    region: lim});
    
/// Exportar puntos de entrenamiento a Google Drive 
     Export.table.toDrive({
     collection: ptos_todos,
     description: 'Exportar_ptos_Entrenamiento',
     fileNamePrefix: 'Puntos_entrenamiento',
     folder: 'Diplomado_GEE',
     fileFormat: 'SHP',
     });
     
  /// Exportar exactitud
    /// Preparación datos
    
    var sta = ee.Feature(null,{
      CM : confusionMatrix,
      PG : pg,
      PU : pu.toList(),
      PP : pp.toList(),
      kappa : k
    }); 
    
    var accuracy = ee.FeatureCollection(sta);
    
    //// Exportar tabla de exactitudes a Google Drive
    Export.table.toDrive({
      collection: accuracy,
      description: 'Exportar_precision',
      folder:'Diplomado_GEE',
      fileNamePrefix:'Precision', 
      fileFormat: 'GeoJSON'
    });
     
     
 
