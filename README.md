# Route 53 Active/Passive Failover

Demonstrate how to configure Route 53 Active/Passive Failover.
Health Check monitors Primary server (e.g. "primary.example.com").
Pair of CNAME records for main website (e.g. "www.example.com") point
to Primary web server (e.g. "primary.example.com") and Secondary
Redirect website (e.g. "redirect.example.com").

Route 53 will normally return the Primary record. If the Health Check
fails, Route 53 will return the Secondary record instead.

# DNS FAILOVER NOTES

Be aware that DNS failover is NOT instant! When the Primary website
fails, failover can take several minutes during which time visitors
will receive an error when trying to visit the Primary website.

* Trigger DNS failover by visiting sample Primary site
([primary.sgf-aws.org](primary.sgf-aws.org)) and toggling the
"Change Page Status" link between OK (200) and Forbidden (403).

* Monitor DNS failover by visiting sample Live site
([site.sgf-aws.org](site.sgf-aws.org)). When failover occurs, you will
be redirected to a Secondary site ([secondary.sgf-aws.org](secondary.sgf-aws.org))
with a link back to the Live site.

* Route 53 DNS records failover in 75-90 seconds. Potentially decrease
this time to 25-30 seconds by changing Health Check interval from
30 seconds to 10 seconds for additional $1/mo.

* DNS servers propagate changes in about 60 seconds due to TTL of
60 seconds. Decreasing TTL further is not recommended.

* CloudFront and S3 are configured to prevent web browser from caching
the redirect file. If web browser receives redirect during failover,
web browser will download new redirect file each visit.

* Web Browsers cache DNS records, which cause them to continue to
retrieve redirect page for up to 15 minutes after DNS points back to
primary website IP. This issue affects Safari, Firefox, Chrome, and
likely other browsers on OSX and likely other OS.

# PREREQUISITES

* Primary Website MUST be accessible over HTTPS via a Common Domain Name
(e.g. www.example.com) that will be pointed to an alternate host during
failover and over HTTPS via an Alternate Domain Name (e.g.
primary.example.com) that AWS Route 53 Health Checks can monitor. This
allows Health Checks to continue monitoring Primary Website during failover.

* The Common Domain Name should have a LOW TTL (e.g. 60 seconds or 300
seconds) so clients will quickly recognize the record change. The
Alternate Domain Name should rarely change and should have a higher TTL
(e.g. 3600 seconds or 86400 seconds).

*  DNS Record for Primary Website (e.g. www.example.com) must be hosted
in AWS Route 53. Note: If you use a 3rd party service (e.g. domain
registrar) to redirect "example.com" to "www.example.com", you could
host "www.example.com" as a DNS Zone without moving the entire domain.

# STATIC SITES

Follow these instructions to prepare the static sites we will use for
our Active/Passive website DNS failover.

### Request Public Certificates

We need to request Public Certificates for the "Primary" and "Secondary"
websites. Note that the certificates must both contain the common site
name. It is recommended that they also contain a unique site name you
can use to browse to each website separately for testing purposes.

AWS Console > [ACM](https://console.aws.amazon.com/acm/home?region=us-east-1#/)

Request **Redirect Certificate** with these Website Domain Names:
```
redirect.sgf-aws.org
site.sgf-aws.org
www.site.sgf-aws.org
```

Request **Secondary Certificate** with these Website Domain Names:
```
secondary.sgf-aws.org
```

Note: During testing, choose hostnames that belong to a domain hosted
by AWS Route 53. This allows you to perform quick DNS validation when
creating each certificate. When the validation page appears, expand
each hostname and click "Create record in Route 53" button for each
hostname. Validation should complete within 3-5 minutes.


### Wait for certificate validation to complete. Make note of each certificate ARN

Redirect
```
arn:aws:acm:us-east-1:347103861163:certificate/61a05d38-8002-4481-aadb-10dee67e213f
```

Secondary
```
arn:aws:acm:us-east-1:347103861163:certificate/b88b7971-3af8-479d-a937-3761c899bf74
```

### Create CloudFormation Stack for Static Websites

This stack will create a Primary and Secondary Bucket in S3. This stack
will also create a Primary and Secondary CloudFront Endpoint for
serving S3 content.

AWS Console > [CloudFormation](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks)

1. Create Stack

    ```
    Upload Template File:
    static-websites-with-cloudfront.yml

    NEXT
    ```

1. Input Stack Parameters

    ```
    Stack Name: static-sites

    Redirect Certificate
    arn:aws:acm:us-east-1:347103861163:certificate/61a05d38-8002-4481-aadb-10dee67e213f

    Secondary Certificate
    arn:aws:acm:us-east-1:347103861163:certificate/b88b7971-3af8-479d-a937-3761c899bf74

    NEXT
    ```

1. Review Stack Options
    ```
    Accept Defaults

    NEXT
    ```

1. Review
    ```
    CREATE STACK
    ```

    Wait 15-30 minutes for this to complete. It can take a LONG time for
    CloudFront distributions to propagate.

1. Stack Outputs

    ```
    When complete, you can find the following details on the Stack "Output" tab.

    PrimaryBucketName	sample-sites-primarybucket-2ybi9yba1ptt
    PrimaryCloudFrontEndpoint	dmy35ict738p0.cloudfront.net
    PrimaryDomainName	primary.sgf-aws.org

    SecondaryBucketName	sample-sites-secondarybucket-b94mlb6kmvpl
    SecondaryCloudFrontEndpoint	d1sf2tky6jmek4.cloudfront.net
    SecondaryDomainName	secondary.sgf-aws.org
    ```

### Upload Website Index Pages

AWS Console > [S3](https://s3.console.aws.amazon.com/s3/home?region=us-east-1) > BUCKET > Upload

Manually Upload "index.html" to Redirect and Secondary S3 bucket via AWS Console.
See Stack Outputs for bucket names. Accept default settings during upload.

```
Redirect
website-redirect/index.html
https://www.w3docs.com/snippets/html/how-to-redirect-a-web-page-in-html.html

Secondary
website-secondary/index.html
"This is the Secondary Website"
```

Afterwards, add the following metadata to the Redirect index.html so that browsers will not cache redirect file.

```
Cache-Control: no-cache, no-store, must-revalidate
Expires: 0
```

NOTE: For simplicity, we use an HTML file with caching disabled to perform the redirect to the secondary site. If you prefer S3 redirects, you must make S3 Redirect Bucket Public and configure CloudFront to use the Website Endpoint instead of the REST Endpoint

```
REST #1: bucket-name.s3.amazonaws.com
REST #2: bucket-name.s3.region.amazonaws.com
Website: bucket-name.s3-website.region.amazonaws.com
```

Reference

https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteEndpoints.html


# DNS FAILOVER

Follow these instructions to create a DNS Health Check, a pair of DNS
records for the Primary Domain Name that point to the Active and Passive
sites.

If Health Check is healthy, Route 53 will return the PRIMARY record.

If Health Check is unhealthy, Route 53 will return the SECONDARY record.

### Manually Configure DNS records pointing to CloudFront Endpoints

Find your Redirect and Secondary CloudFront Endpoints in your Stack
Output. Create a CNAME with a TTL of 3600 seconds (1 hour) that points
each public hostname to the relevant CloudFront Endpoint

AWS Console > Route 53 > Hosted Zones > ZONE > Create Record Set

```
Redirect
redirect.sgf-aws.org
d20nucmzhhtllp.cloudfront.net

Secondary
secondary.sgf-aws.org
d2lbxjt5l9nwd.cloudfront.net
```

### Browse to Static Websites

Verify the redirect URL immediately redirects you to the Secondary URL.
Verify the Secondary URL displays "This is the Secondary Website".

```
Redirect
https://redirect.sgf-aws.org

Secondary
https://secondary.sgf-aws.org
```

### Create CloudFormation Stack for DNS Health Check and Record Sets

This stack will create a DNS Health Check, a pair of DNS Records for
the Primary Domain (e.g. site.sgf-aws.org) that point to the Active
(e.g. primary.sgf-aws.org) and Passive (e.g. redirect.sgf-aws.org) sites.

AWS Console > CloudFormation

1. Create Stack

    ```
    Upload Template File:
    dns-failover-active-passive.yml

    NEXT
    ```

1. Input Stack Parameters

    ```
    Stack Name:
    dns-failover

    Hosted Zone Name
    sgf-aws.org

    Primary Domain Name
    site.sgf-aws.org

    Active Domain Name
    primary.sgf-aws.org

    Passive Domain Name
    redirect.sgf-aws.org

    NEXT
    ```

1. Review Stack Options
    ```
    Accept Defaults

    NEXT
    ```

1. Review
    ```
    CREATE STACK
    ```

1. Stack Outputs

    When complete, you can find the following details on the Stack "Output" tab.

    ```
    ActiveDomainName	primary.sgf-aws.org
    PassiveDomainName	redirect.sgf-aws.org
    PrimaryDomainName	site.sgf-aws.org
    ```

Reference

https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html#dns-failover-types-active-passive-one-resource

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-route53-recordset.html

https://docs.amazonaws.cn/en_us/AWSCloudFormation/latest/UserGuide/quickref-route53.html#scenario-route53-recordset-by-host


## Authors

* **Jason Klein** - *Initial work* - [jason-klein](https://github.com/jason-klein)

See also the list of [contributors](https://github.com/jason-klein/route53-active-passive-failover/contributors) who participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments

* Thank you to [Missouri State University](https://www.missouristate.edu/) for partially funding this research.
