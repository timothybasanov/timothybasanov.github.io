## Wait for docker is healthy

```sh
{% raw %}
while true; do
  STATUS="$( ( docker-compose ps -q ; echo "container-name" ) | xargs docker inspect --format \
      '{{.Name}}={{.State.Status}}/{{with .State.Health}}{{.Status}}{{end}}' )"
  if [[ ! "$(echo "${STATUS}" | grep 'unhealthy' | wc -l)" -eq 0 ]]; then
    echo "[WARNING] A docker container is unhealthy: $(echo "${STATUS}" | xargs echo)" >&2
    exit 1
  elif [[ ! "$(echo "${STATUS}" | grep 'starting' | wc -l)" -eq 0 ]]; then
    echo "[INFO] A docker containers is starting: $(echo "${STATUS}" | xargs echo)" >&2
  elif [[ ! "$(echo "${STATUS}" | grep -v 'running' | wc -l)" -eq 0 ]]; then
    echo "[INFO] A docker container is not running: $(echo "${STATUS}" | xargs echo)" >&2
  else
    # Either healthy or health check was not defined
    echo "[INFO] All docker containers are healthy: $(echo "${STATUS}" | xargs echo)" >&2
    break;
  fi
  sleep 1
done
{% endraw %}
```
