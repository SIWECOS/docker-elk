ARG ELK_VERSION
ARG DEFAULT_PASSWORD

# https://github.com/elastic/elasticsearch-docker
FROM docker.elastic.co/elasticsearch/elasticsearch:${ELK_VERSION}

# Set custom elasticseach password
# https://stackoverflow.com/a/55599206/11215251
RUN echo "changeme"| bin/elasticsearch-keystore add -xf bootstrap.password

# Add your elasticsearch plugins setup here
# Example: RUN elasticsearch-plugin install analysis-icu
