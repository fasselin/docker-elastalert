# Docker ElastAlert
Docker container for [Yelp's ElastAlert](https://github.com/Yelp/elastalert).

## Configuration
The config.yaml file will be used as configuration, added to the container during the building step. Some configuration values will be replaced by environment variables while the container is running.

## Building
The rules defined in the `rules` folder will be added to the ElastAlert container on build time, so if you want to change your rules, a new version of the container must be built.

You can build the container like

```bash
$ docker build -t fiunchinho/docker-elastalert .
```

## Running
This container needs two environment variables when is running

- `ELASTICSEARCH_HOST`: ElasticSearch host to query.
- `ELASTICSEARCH_PORT`: ElasticSearch port (Default: 9200).
- `AWS_REGION`: AWS Region to use.
- `USE_SSL`: Use ssl (Default: False)
- `SNS_TOPIC_ARN`: The ARN of the SNS topic to publish to.
- `AUTH_METHOD`: Authentication method. Either `boto_profile` or `instance_role` 
- `BOTO_PROFILE`: Boto profile to use to connect to AWS.

So you can start this container like

```bash
$ docker run -e "ELASTICSEARCH_HOST=some.elasticsearch.host.com" -e "ELASTICSEARCH_PORT=9200" -e "AWS_REGION=eu-west-1" -e "AUTH_METHOD=instance_role" fiunchinho/docker-elastalert
```

## Running against Amazon ElasticSearch service
Since Amazon ElasticSearch service doesn't provide a way to secure your ElasticSearch using network firewall rules, we need to sign the requests to ElasticSearch. There two different mechanism to sign requests.

### Using instance role
When you deploy an EC2 instance to AWS, you assign a specific role to the instance. That role must have read/write permissions with ElasticSearch. In this case you need to pass these environment variables
- `AUTH_METHOD`: `instance_role`
- `AWS_REGION`: Region to connect

### Using a boto profile
If you want to execute this docker container locally, you can use a boto profile to sign your requests to ElasticSearch. To do that, you have to mount your `credentials` folder inside the container and **set the aws_region and boto_profile parameter in both the `config.yml` file and your rule file**. Then you need to pass these environment variables
- `AUTH_METHOD`: `boto_profile`
- `AWS_REGION`: Region to connect
- `BOTO_PROFILE`: The profile to use, from the `~/.aws/credentials` file

For example

```bash
$ docker run -v "$HOME/.aws:/root/.aws" -e "ELASTICSEARCH_HOST=some.elasticsearch.host.com" -e "ELASTICSEARCH_PORT=9200" -e "AUTH_METHOD=boto_profile" -e "AWS_REGION=eu-west-1" -e "BOTO_PROFILE=preproduction" fiunchinho/docker-elastalert
```

## Alerting
Depending on your desired alerts you may need to mount files into the container, like AWS credentials for SNS alerting or smtp configuration values for Email alerting.

### Email
Alerts using email need to specify the path to a file which contains SMTP authentication credentials. So you need to mount this file inside the container. If the file `email_credentials.yml` is inside your current folder and your rule expect it to be in `/tmp/email_credentials.yml`

```bash
$ docker run -v "$PWD/email_credentials.yml:/tmp/email_credentials.yml" -e "ELASTICSEARCH_HOST=some.elasticsearch.host.com" -e "ELASTICSEARCH_PORT=9200" -e "AWS_REGION=eu-west-1" -e "AUTH_METHOD=instance_role" fiunchinho/docker-elastalert
```

### SNS
For example, if we want to alert using SNS we need to specify a SNS topic using the environment variable `SNS_TOPIC_ARN`, and make sure that we use a `boto_profile` or `instance_role` with permissions to publish in the SNS topic

```bash
$ docker run -e "ELASTICSEARCH_HOST=some.elasticsearch.host.com" -e "ELASTICSEARCH_PORT=9200" -e "SNS_TOPIC_ARN=arn:aws:sns:us-west-1:112233" -e "AWS_REGION=eu-west-1" -e "AUTH_METHOD=instance_role" fiunchinho/docker-elastalert
```

## FAQ
### Container just hangs with no output. What should I do?
This happens when the requests from ElastAlert can't be authenticated. If running locally using `boto_profile`, check that you've **set the aws_region and boto_profile parameter in both the `config.yml` file and your rule file** and the credentials file is mounted on the container. If you are using `instance_role` instead of `boto_profile`, most likely the role assigned to the server has no the right permissions to access Amazon ElasticSearch service.