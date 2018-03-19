# https://github.com/elastic/logstash-docker
FROM docker.elastic.co/logstash/logstash:6.2.2
RUN logstash-plugin install logstash-input-beats
RUN logstash-plugin install logstash-input-log4j
# Add your logstash plugins setup here
# Example: RUN logstash-plugin install logstash-filter-json