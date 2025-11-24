ARG ELASTIC_VERSION

# https://www.docker.elastic.co/
FROM docker.elastic.co/elasticsearch/elasticsearch:${ELASTIC_VERSION:-9.2.1}
USER root
COPY ./config/elasticsearch.yml /usr/share/elasticsearch/config/elasticsearch.yml
# Add your elasticsearch plugins setup here
# Example: RUN elasticsearch-plugin install analysis-icu
