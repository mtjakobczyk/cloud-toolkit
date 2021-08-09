Lookup management events (control plane API calls) since `2021-08-01` related to the attribute `ResourceType` with value `AWS::EC2::Instance` and perform client-side filtering:
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::Instance \
  --start-time 2021-08-01 \
  --output=json --query $QUERY
```

Sample client-side filters:
```bash
QUERY="Events[?Resources[?ResourceName == 'i-0z13c56456456549mm3f87']].{e: EventName, res: Resources}"
```
