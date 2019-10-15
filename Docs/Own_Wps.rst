Create your own WPS
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Presentation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The OGC Web Processing Service (WPS) Interface Standard provides rules for standardizing inputs and outputs (requests and responses) for invoking geospatial processing services as a web service.

WPS scripts for GeoServer are written in groovy language. They should be located in the Geoserver\\data_dir\\scripts\\wps directory.

.. tip::
    The best way to make your own WPS is to be inspired by those that are already made. See how the tutorial is built or contact us for many more examples.

General Structure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1.Import used libraries
-------------------------

::

    import geoserver.GeoServer
    import geoserver.catalog.Store



2.WPS Script meta data
-------------------------

::

    title = '....'
    description = '.....'

3.WPS Script input & output
-----------------------------------

::

    inputs = [
        inputparameter1: [name: '...', description : '...', title: '...', type: String.class],
        inputparameter2: [name: '...', description : '...', title: '...', type: String.class]
    ]

    outputs = [
        ouputparameter: [name: '...', title: '...', type: String.class]
    ]

4.Set connection method
-----------------------------------

::

    def static Connection openPostgreSQLDataStoreConnection() {
        Store store = new GeoServer().catalog.getStore("h2gisdb")
        JDBCDataStore jdbcDataStore = (JDBCDataStore)store.getDataStoreInfo().getDataStore(null)
        return jdbcDataStore.getDataSource().getConnection()
    }



5.Set main method to execute 
-----------------------------------

::

    def run(input) {
        // Open connection
        Connection connection = openPostgreSQLDataStoreConnection()

        // Execute code
        // Here

        // print to Console windows
        return [result : 'Ok ! ']
    }

Run SQL command line
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Example : 

::

    Sql sql = new Sql(connection)
    sql.execute("drop table if exists TABLETODROP")    


Init Noise Modelling
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    PointNoiseMap pointNoiseMap = new PointNoiseMap(building_table_name, sources_table_name, receivers_table_name)
    pointNoiseMap.setComputeHorizontalDiffraction(compute_horizontal_diffraction)
    pointNoiseMap.setComputeVerticalDiffraction(compute_vertical_diffraction)
    pointNoiseMap.setSoundReflectionOrder(reflexion_order)
    // Building height field name
    pointNoiseMap.setHeightField("HEIGHT")
    // Import table with Snow, Forest, Grass, Pasture field polygons. Attribute G is associated with each polygon
    //pointNoiseMap.setSoilTableName(ground_table_name);
    // Point cloud height above sea level POINT(X Y Z)
    //pointNoiseMap.setDemTable(dem_table_name);
    // Do not propagate for low emission or far away sources.
    // error in dB
    pointNoiseMap.setMaximumError(0.1d);
    pointNoiseMap.setMaximumPropagationDistance(max_src_dist)
    pointNoiseMap.setMaximumReflectionDistance(max_ref_dist)
    pointNoiseMap.setWallAbsorption(wall_alpha)
    pointNoiseMap.setThreadCount(n_thread)

    pointNoiseMap.initialize(connection, new EmptyProgressVisitor());



Run Noise Modelling
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    // Iterate over computation areas
    for (int i = 0; i < pointNoiseMap.getGridDim(); i++) {
        for (int j = 0; j < pointNoiseMap.getGridDim(); j++) {
            // Run ray propagation
            IComputeRaysOut out = pointNoiseMap.evaluateCell(connection, i, j, progressVisitor, receivers);
            // Return results with level spectrum for each source/receiver tuple
            if(out instanceof ComputeRaysOut) {
                ComputeRaysOut cellStorage = (ComputeRaysOut) out;
                allLevels.addAll(((ComputeRaysOut) out).getVerticesSoundLevel())
                cellStorage.receiversAttenuationLevels.each { v -> 
                    double globalDbValue = ComputeRays.wToDba(ComputeRays.sumArray(ComputeRays.dbaToW(v.value)));
                    def idSource = out.inputData.SourcesPk.get(v.sourceId)
                    double[] w_spectrum  = ComputeRays.wToDba(out.inputData.wjSourcesD.get(idSource))
                    SourceSpectrum.put(v.sourceId as Integer,w_spectrum)
                }
            }
        }
    }