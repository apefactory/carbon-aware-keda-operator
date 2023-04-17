# Carbon Aware KEDA Operator

This repo provides a Kubernetes operator that aims to reduce carbon emissions by helping [KEDA](https://keda.sh/) scale Kubernetes workloads based on carbon intensity. Carbon intensity is a measure of how much carbon dioxide is emitted per unit of energy consumed. By scaling workloads according to the carbon intensity of the region or grid where they run, we can optimize the carbon efficiency and environmental impact of our applications.

This operator can use carbon intensity data from third party sources such as [WattTime](https://www.watttime.org/), [Electricity Map](https://www.electricitymap.org/) or any other provider, to dynamically adjust the scaling behavior of KEDA. The operator does not require any application or workload code change, and it works with any KEDA scaler.

Use cases for the operator include low priority and time flexible workloads that support interuptions in dev/test environments. Some examples of these are non-critical data backups, batch processing jobs, data analytics processing, and ML training jobs.

To read more about carbon intensity and carbon awareness, please check out this course from [Green Software Foundation](https://learn.greensoftware.foundation/carbon-awareness/).

## How it works


![image](https://user-images.githubusercontent.com/966110/232321762-bb114716-f719-4e1f-9dc0-20680c94ab52.png)



 #### Getting the carbon intensity data:

The carbon aware KEDA operator retrieves the carbon intensity data from a ConfigMap, which is generated by a third party component. 

In our demo, we used the [Kubernetes Carbon Intensity Exporter](https://github.com/Azure/kubernetes-carbon-intensity-exporter/) operator, which builds on the [carbon-aware-sdk](https://github.com/Green-Software-Foundation/carbon-aware-sdk), to provide carbon intensity data in the Kubernetes cluster, so it can be used by operators for carbon aware decision making. 

The "Kubernetes carbon intensity exporter" retrieves 24-hour carbon intensity forecast data every 12 hours. Upon successful data pull, the old configmap will be deleted and a new configmap with the same name will be created. 

Any other Kubernetes operator or workload can read the configMap for utilizing the carbon intensity data.

<br>

 #### Making carbon aware scaling decisions:

As an admin you create a CarbonAwareKedaScaler spec for targetRef : scaledObject or scaledJob

Then the operator will update KEDA scaledObjects and scaledJob `maxReplicaCount` field, based on the current carbon intensity.



## Current carbon aware scaling logic 

The current logic for carbon aware scaling is based on carbon intensity metric only, _which is independent of the workload usage_.

The operator will not compute a desired replicaCount for your scaledObjects or scaledJobs, as this is the responsibility of KEDA and HPA. The operator would define a "ceiling for allowed maxReplicas" based on carbon intensity of the current time.

In practice, this operator will throttle workloads and prevent them from bursting during high carbon intensity periods, and allow more scaling when carbon intensity is lower.

## How to use it

Once the "carbon aware KEDA operator" is installed, you can deploy a custom resource called `CarbonAwareKedaScaler` to set the max replicas, KEDA can scale up to, based on carbon intensity.

The `CarbonAwareKedaScaler` CRD defines the following settings:

- The `carbonIntensityForecastDataSource` field specifies the data source for carbon intensity forecast data and can be set to either use mock carbon forecast data or a configmap for carbon forecast data. 

- The `maxReplicasByCarbonIntensity` field specifies an array of carbon intensity values in ascending order; each threshold value represents the upper limit and previous entry represents lower limit. When carbon intensity is below a certain threshold value, more replicas are created and when it’s above a certain threshold value, fewer replicas are created. 

- The `ecoModeOff` field contains settings to disable carbon awareness; it can be overriden based on high intensity duration or time schedules.


```yaml
apiVersion: carbonaware.kubernetes.azure.com/v1alpha1 
kind: CarbonAwareKedaScaler 
metadata: 
  name: carbon-aware-word-processor-scaler
spec: 
  kedaTarget: scaledobjects.keda.sh        # can be used for ScaledObjects & ScaledJobs
  kedaTargetRef: 
    name: word-processor-scaler
    namespace: default 
  carbonIntensityForecastDataSource:       # carbon intensity forecast data source 
    mockCarbonForecast: false              # [OPTIONAL] use mock carbon forecast data 
    localConfigMap:                        # [OPTIONAL] use configmap for carbon forecast data 
      name: carbon-intensity 
      namespace: kube-system
      key: data 
  maxReplicasByCarbonIntensity:            # array of carbon intensity values in ascending order; each threshold value represents the upper limit and previous entry represents lower limit 
    - carbonIntensityThreshold: 437        # when carbon intensity is 437 or below 
      maxReplicas: 110                     # do more 
    - carbonIntensityThreshold: 504        # when carbon intensity is >437 and <=504 
      maxReplicas: 60 
    - carbonIntensityThreshold: 571        # when carbon intensity is >504 and <=571 (and beyond) 
      maxReplicas: 10                      # do less 
  ecoModeOff:                              # [OPTIONAL] settings to override carbon awareness; can override based on high intensity duration or schedules 
    maxReplicas: 100                       # when carbon awareness is disabled, use this value 
    carbonIntensityDuration:               # [OPTIONAL] disable carbon awareness when carbon intensity is high for this length of time 
      carbonIntensityThreshold: 555        # when carbon intensity is equal to or above this value, consider it high 
      overrideEcoAfterDurationInMins: 45   # if carbon intensity is high for this many hours disable ecomode 
    customSchedule:                        # [OPTIONAL] disable carbon awareness during specified time periods 
      - startTime: "2023-04-28T16:45:00Z"  # start time in UTC 
        endTime: "2023-04-28T17:00:59Z"    # end time in UTC 
    recurringSchedule:                     # [OPTIONAL] disable carbon awareness during specified recurring time periods 
      - "* 23 * * 1-5"                     # disable every weekday from 11pm to 12am UTC 
```

## Format of the input ConfigMap

The [generated carbon intensity configMap](https://github.com/Azure/kubernetes-carbon-intensity-exporter#integration) has the following format:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: carbonintensity
  namespace: kube-system
immutable: true
data:
  lastHeartbeatTime: # The latest time that the data exporter controller sends the data. 
  message: # Additional information for user notification, if any. 
  numOfRecords: # The number can be any value between 0 (no records for the current location) and 24 * 12. 
  forecastDateTime: # The time when the raw data was generated.
  minForcast: # min forecast in the data.
  maxForcast: # max forecast in the data.
binarydata: 
  data: # json marshal of the EmissionsData array.
```


## Installation & demo

To install the Carbon Aware KEDA Operator, please check out the following links.
-	[Install on AKS](/demo/azure.md)
-	[Install on Kind](/demo/kind.md)

## FAQ

### How do I set the carbon intensity thresholds in the `CarbonAwareKedaScaler` CRD

When adding `maxReplicasByCarbonIntensity` entries in the custom resource, it is important to understand what the carbon intensity values are, since they vary between regions. 


The carbon intensity ConfigMap provides minimum and maximum carbon intensity values, to help you set thresholds accordingly.`
> The configMap will show min & max carbon intensity forecasted values, for the next 24h, 72h, next week...depending on the data source provider you use. 
> To have a more accurate min & max carbon intensity values, you should look at monthly or yearly data from your carbon intensity provider.

```
#ConfigMap

data:
  minForcast: 370 # min forecast in the data.
  maxForcast: 571 # max forecast in the data.
```

> Remember, when energy is dirty (e.g., carbon intensity is high), do less, and when energy is clean (e.g., carbon intensity is low), do more.

```
#CarbonAwareKedaScaler

maxReplicasByCarbonIntensity:            # array of carbon intensity values in ascending order; each threshold value represents the upper limit and previous entry represents lower limit 
    - carbonIntensityThreshold: 437        # when carbon intensity is 437 or below 
      maxReplicas: 110                     # do more 
    - carbonIntensityThreshold: 504        # when carbon intensity is >437 and <=504 
      maxReplicas: 60 
    - carbonIntensityThreshold: 571        # when carbon intensity is >504 and <=571 (and beyond) 
      maxReplicas: 10                      # do less 
```

To set the thresholds, the idea is to find ranges between minimum and maximum carbon intensity and divide them into “buckets”. 

In the example above, we use 3 thresholds that represent “low”, “medium”, and “high” buckets where :

- the 3 buckets size is defined by : (max - min) / 3 = (571 - 370) / 3 = 67

- low bucket : carbon intensity is <= 437 (= 370 + 67),

- medium bucket : carbon intensity is > 437 and <= 504 (= 370 + 67 + 67),

- high bucket : carbon intensity is > 504 and <= 571 (Or higher > than 571, since this is the highest threshold defined in the array)

Configuring thresholds in an array like this gives you flexibility to create as many thresholds/buckets as needed.

### How do I set the allowed maxReplicas per carbon intensity in the `CarbonAwareKedaScaler` CRD
 
It’s up to you as an admin or a developer, to decide the carbon aware scaling behavior for your workload:

- You could decide to use only one carbon intensity thresholds or several (such as low, medium, high buckets)
- You could scale to zero during high carbon intensity periods, or keep a minimal replicas running for your workload.
- Depending on the nature of the workload and its constraints, you would decide what scaling limits are suitable for you workload.

### What metrics are exported by the operator? 

The following metrics are exported by the operator:

- `carbon_intensity`: The carbon intensity of the electricity grid region where Kubernetes cluster is deployed
- `MaxReplicas`: The maximum number of replicas that can be scaled up to by the KEDA scaledObject or scaledJob, based on carbon intensity.
- `Default MaxReplicas`: The default value of `MaxReplicas` when carbon awanress is disabled, aka "ecoMode off".


## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
