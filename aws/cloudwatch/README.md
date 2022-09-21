#### Create Log Group
```
LG_NAME="/app/mock/application"
TAGS="environment=dev,assetOwner=Michal"
aws logs create-log-group --log-group-name $LG_NAME --tags "$TAGS"
aws logs put-retention-policy --log-group-name $LG_NAME --retention-in-days 3
```

#### Publish sample logs to a Log Stream
```
LG_NAME="/app/mock/application"
LS_NAME="stream-$(date '+%Y%m%d')-$(uuid)"

aws logs create-log-stream --log-group-name $LG_NAME --log-stream-name $LS_NAME

LEN=3

generateLogMessages()
{
for i in $(seq 1 $LEN);
do
  TS=$(($(date +%s%N)/1000000))
  MSG=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c10)
  echo -n "timestamp=$TS,message=$MSG "
done
}

aws logs put-log-events --log-group-name $LG_NAME --log-stream-name $LS_NAME --log-events $(generateLogMessages)
```
