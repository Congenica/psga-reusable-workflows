# Start runners using EC@ fleet requests

The idea is to replace 
- [ec2-runner-start.yaml](.github/workflows/ec2-runner-start.yaml)
- [ec2-runner-stop.yaml](.github/workflows/ec2-runner-stop.yaml)

so it will be possible to start the runners in different AZs and use a list of allowed instance types instead of one particular instance.

### Configuration

use AWS CLI to create the fleets of 1 instance capacity (spot or on-demand)

use fleet.json file to specify a launch template configuration and network specification.


### Calling Start Runner Workflow

```json
  {
    "LaunchTemplateConfigs": [
    {
        "LaunchTemplateSpecification": {
        "LaunchTemplateId": "lt-052396bfbe3244c59", // this LT is mine, need to replace with GH runner LT
        "Version": "$Latest"
        },
        "Overrides": [
                {
                    "SubnetId": "subnet-027b983fd1e480593,subnet-0ecea06d86ddf6fb6",
                    "InstanceRequirements": {
                        "ExcludedInstanceTypes": [],
                        "AllowedInstanceTypes": ["c5.large", "c5a.large", "c6i.large", "c6a.large", "c7i.large"],
                        "VCpuCount": {
                            "Min": 2,
                            "Max": 4
                        },
                        "MemoryMiB": {
                            "Min": 3072,
                            "Max": 8192
                        }
                    }
                }
            ]
    }
    ],
    "TargetCapacitySpecification": {
    "TotalTargetCapacity": 1,
    "DefaultTargetCapacityType": "spot"
    }
  }

```

### Calling Start workflow

```bash
   aws ec2 create-fleet --type request --cli-input-json file://fleet.json
```

### Output
```json
   {
    "FleetId": "fleet-19aed864-031c-4610-9567-a3edd78732ca"
   }
```
---


### Calling Stop workflow
```bash
   aws ec2 delete-fleets --fleet-ids fleet-19aed864-031c-4610-9567-a3edd78732ca --terminate-instances
```

### Output
```json
{
    "SuccessfulFleetDeletions": [
        {
            "CurrentFleetState": "deleted_terminating",
            "PreviousFleetState": "active",
            "FleetId": "fleet-19aed864-031c-4610-9567-a3edd78732ca"
        }
    ],
    "UnsuccessfulFleetDeletions": []
}
```

#### Docs
- [which-spot-request-method-to-use](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html#which-spot-request-method-to-use)
- [API_CreateFleet](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_CreateFleet.html)
- [ec2-fleet-request-types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-request-type.html)
