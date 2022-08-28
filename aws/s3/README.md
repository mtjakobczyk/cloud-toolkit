### List Public Access Block configuration for each bucket

    for bn in $(aws s3api list-buckets --output text --query Buckets[].Name); 
    do 
      echo "BUCKET: $bn"; 
      aws s3api get-public-access-block --bucket $bn; 
      echo "###"; 
    done
