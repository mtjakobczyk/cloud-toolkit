### List Public Access Block configuration for each bucket

    for bn in $(aws s3api list-buckets --output text --query Buckets[].Name); 
    do 
      echo "BUCKET: $bn"; 
      aws s3api get-public-access-block --bucket $bn; 
      echo "###"; 
    done

### List S3 Buckets with no Public Access Block configuration

    for bn in $(aws s3api list-buckets --output text --query Buckets[].Name); 
    do 
      aws s3api get-public-access-block --bucket $bn > /dev/null 2>&1;
      if [ "$?" -ne 0 ]; then
        echo "Missing Public Access Block configuration: $bn";
      fi
    done

### Display Bucket ACL for S3 Buckets with no Public Access Block configuration

    for bn in $(aws s3api list-buckets --output text --query Buckets[].Name); 
    do 
      aws s3api get-public-access-block --bucket $bn > /dev/null 2>&1;
      if [ "$?" -ne 0 ]; then
        echo "BUCKET: $bn";
        echo "Missing Public Access Block configuration";
        aws s3api get-bucket-acl --bucket $bn;
        echo "####################";
      fi
    done
