# Habitat Suitability Modeling of Vermont Beavers
Utilize Python to script analytics with ArcGIS Pro, and conduct a habitat suitability model of beavers in Vermont.

# 1: Initialize your ArcGIS Pro Environment

```r
aprx = arcpy.mp.ArcGISProject(os.getcwd() + "\\SuitabilityModeling.aprx")
arcpy.env.workspace = aprx.homeFolder + "\\Data.gdb"
arcpy.env.scratchWorkspace = "in_memory"
print("Project not known, home folder here: \n" + aprx.homeFolder)
print("Current workspace: \n" + arcpy.env.workspace )
arcpy.env.overwriteOutput = True
```

 # 2: Specify Required Layers

 ```r
streams           = "Streams"
elevation         = "Elevation"
landuse           = "LandUse"
food_submodel     = "FoodSubmodel"
security_submodel = "SecuritySubmodel"
exist_regions     = "ExistingRegion"
costsurface       = "CostSurface"
```

# 3: Calculate Slope & Distance from Streams

```r
# 1. Calculate Slope
slope = Slope(elevation, "DEGREE", 1)

# 2. Calculate Distance to Streams
dist_streams = EucDistance(streams, cell_size=30)
```

# 4: Scale & Reclassify Layers

```r
landuse_recl       = Reclassify("landuse", "Value", "1 1;2 1;3 2;4 3;5 4;6 9;7 7;8 8;9 10;10 9;11 2;12 1", "DATA")
slope_trans        = RescaleByFunction(slope, TfLogisticGrowth(), from_scale=1, to_scale=10)
dist_streams_trans = RescaleByFunction(dist_streams, TfMSSmall(), from_scale=1, to_scale=10)
```

# 5: Calculate Weighted Sum Habitat Suitability Model

```r
Habitat_WSumTableObj = WSTable([[landuse_recl,       "VALUE", 2],
                        [slope_trans,        "VALUE", 1],
                        [dist_streams_trans, "VALUE", 1]])
habitat_suitability  = WeightedSum(Habitat_WSumTableObj)
habitat_suitability.save(os.path.join(arcpy.env.workspace, "HabitatSuitabilityModel"))
```
![HSM](https://raw.githubusercontent.com/JTSALAH/Beaver-Habitat-Suitability_Modeling/main/IMAGES/HSM.png)
# 6: Combine Habitat Suitability Model with Food & Security Models

```r
Suitability_WSumTableObj = WSTable([[habitat_suitability, "VALUE", 1],
                                    [food_submodel,       "VALUE", 1.5],
                                    [security_submodel,   "VALUE", 1.5]])
suitability_model = WeightedSum(Suitability_WSumTableObj)
suitability_model.save(os.path.join(arcpy.env.workspace, "SuitabilityModel"))
```
![SM](https://raw.githubusercontent.com/JTSALAH/Beaver-Habitat-Suitability_Modeling/main/IMAGES/SM.png)

# 7: Locate Habitat Patches frrom Suitability Surface

```r
# Locate Habitat Patches from the Suitability Surface
suitable_regions = LocateRegions(suitability_model, 50, "SQUARE_MILES", 5, "CIRCLE",
                                 0, 50, "HIGHEST_AVERAGE_VALUE", 5, 14, 4, 13, "MILES",
                                 exist_regions)

# Ensure Final Suitability Regions Include Existing Regions
exist_regions_recl = Reclassify(exist_regions, "Value", "1 6", "DATA")
suitable_regions_final = CellStatistics([exist_regions_recl, suitable_regions], "MAXIMUM", "DATA")
suitable_regions_final.save(os.path.join(arcpy.env.workspace, "SuitableRegions"))
```
![SR](https://raw.githubusercontent.com/JTSALAH/Beaver-Habitat-Suitability_Modeling/main/IMAGES/SR.png)
# 8: Connect Regions with an Optimum Network

```r
connectivity = CostConnectivity(suitable_regions_final, costsurface, "OptimumNetwork")
connectivity.save(os.path.join(arcpy.env.workspace, "OptimumNetwork"))
```
![SRC](https://raw.githubusercontent.com/JTSALAH/Beaver-Habitat-Suitability_Modeling/main/IMAGES/SRC.png)

