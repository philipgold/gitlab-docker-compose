# Create a docker-compose file for fully running gitlab

Currently there is no way to run the gitlab as a whole with just docker compose. I see that the gitlab and gitlab runner needs to be initialised separately and one needs to contact each other. This can still be solved using this docker compose.



```bash
$ docker network create -d bridge proxy_proxy_default
```





```bash
$ docker-compose up --build
```



* Create 2 containers - gitlab, gitlab-runner using docker compose file.
* Let both containers come up
* The gitlab-runner can wait until the token generated is available in a file
    * User would go in and get the token from gitlab web
    * User would execute a command in second container like
      docker exec -it ea1231df save-token 
* This will cause the second container to continue

I dont see any way to save the tags and others to a file and load it in container. Anything that is available as checkboxes ideally(atleast critical ones) should be configurable in a file. Only this way we can make everything declarative






I automated this using a second service next to the runner itself. That service runs `runner_registration.sh`:

```sh
#!/bin/env bash

# -u especially important: error if environment variable is unset.
set -eu

N_ATTEMPTS=0
# Initial start-up can take a while...
MAX_ATTEMPTS=60

# See also: https://stackoverflow.com/a/50583452/11477374
until $(curl --output /dev/null --silent --head --fail -L -H "Host: ${VIRTUAL_HOST}" ${VIRTUAL_HOST}); do
    if [ ${N_ATTEMPTS} -eq ${MAX_ATTEMPTS} ]
    then
      echo "Maximum number of attempts reached, exiting." >&2
      exit 1
    fi

    echo "Could not reach ${VIRTUAL_HOST}, trying again... (attempt number ${N_ATTEMPTS})"
    N_ATTEMPTS=$(( $N_ATTEMPTS + 1 ))
    sleep 5
done

echo "Reached ${VIRTUAL_HOST} after ${N_ATTEMPTS} tries, attempting to register runner."

gitlab-runner \
    register \
    --non-interactive \
    --url=https://${VIRTUAL_HOST} \
    --registration-token=${INITIAL_RUNNER_TOKEN} \
    --executor=docker \
    --docker-image=debian \
    --description=local

echo "Runner registration successful."
```

The `docker-compose.yml` file is:

```sh
version: "3.8"

volumes:
  config:
  logs:
  data:
  runner_config:

networks:
  # Join the same network as the reverse proxy from a parallel docker compose.
  # See also https://stackoverflow.com/a/38089080/11477374
  # The part before the first underscore of the network name is the parallel
  # docker-compose project name, which defaults to the directory name.
  proxy_default:
    external: true

services:
  web:
    # GitLab Community Edition is same as GitLab Enterprise Edition Core,
    # but fully open source.
    image: gitlab/gitlab-ce
    restart: unless-stopped
    environment:
      VIRTUAL_HOST: ${VIRTUAL_HOST}
      LETSENCRYPT_HOST: ${VIRTUAL_HOST}
      TZ: ${TZ}
      # The `GITLAB_OMNIBUS_CONFIG` holds config settings. It would be nicer to have it
      # in its own file, but having it here helps with keeping the config DRY, using the
      # `.env` file.
      #
      # This instance runs behind a reverse proxy.
      # The `gitlab/gitlab-ce` image contains its own proxy and SSL setup.
      # That is amazing, but not what is required for this setup.
      # To disable it, see:
      # https://docs.gitlab.com/omnibus/settings/nginx.html#supporting-proxied-ssl
      # There, it says to keep the 'https' part of `external_url`, however I had to
      # play around with that a bit, see also:
      # https://forum.gitlab.com/t/gitlab-redirecting-to-https-although-it-is-disabled/18616
      # https://forum.gitlab.com/t/gitlab-using-docker-compose-behind-a-nginx-reverse-proxy/26148
      #
      # For SMTP config, see:
      # https://docs.gitlab.com/omnibus/settings/smtp.html#fastmail
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://${VIRTUAL_HOST}'
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "${SMTP_ADDRESS}"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "${SMTP_USER}"
        gitlab_rails['smtp_password'] = "${SMTP_PASSWORD}"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['smtp_openssl_verify_mode'] = 'peer'
        gitlab_rails['gitlab_email_from'] = 'info@${VIRTUAL_HOST}'
        gitlab_rails['gitlab_email_reply_to'] = 'noreply@${VIRTUAL_HOST}'
        gitlab_rails['time_zone'] = '${TZ}'
        gitlab_rails['initial_root_password'] = '${INITIAL_ROOT_PASSWORD}'
        gitlab_rails['initial_shared_runners_registration_token'] = "${INITIAL_RUNNER_TOKEN}"
    networks:
      - default
      - proxy_default
    volumes:
      - config:/etc/gitlab
      - logs:/var/log/gitlab
      - data:/var/opt/gitlab
  runner:
    image: gitlab/gitlab-runner
    depends_on:
      - web
      - runner_registration
    environment:
      - TZ
    volumes:
      # Give access to Docker for runner to execute jobs.
      - /var/run/docker.sock:/var/run/docker.sock
      - runner_config:/etc/gitlab-runner
  runner_registration:
    # This will register the runner with the GitLab instance, providing a config file
    # for the actual runner to read from. After the config file is created, this service
    # exits.
    # Note that this process may add the same runner over and over across docker-compose
    # restarts. This does not seem to be a problem.
    build:
      context: ./runner
    image: gitlab/gitlab-runner:register
    environment:
      - VIRTUAL_HOST
      - INITIAL_RUNNER_TOKEN
    depends_on:
      - web
    volumes:
      - runner_config:/etc/gitlab-runner
```

where the `Dockerfile` for the `runner_registration` service is simply:

```dockerfile
FROM gitlab/gitlab-runner

COPY ./runner_registration.sh /

RUN chmod +x /runner_registration.sh

ENTRYPOINT [ "/bin/bash", "/runner_registration.sh" ]
```

That is, it runs the above bash script and then exits. Services exiting after only a couple seconds or minutes is probably not the way proper way for Docker compose, but it works. That registration service writes a config (then exits) that is then read from the real runner.

The core setting of all this is [this one](https://gitlab.com/gitlab-org/omnibus-gitlab/-/blob/73718929efdee31e2d1c70047d762144c09409c3/files/gitlab-config-template/gitlab.rb.template#L604):

```sh
gitlab_rails['initial_shared_runners_registration_token'] = "token"
```

This allows us to know the token beforehand, without requiring the GUI. I found this while looking around the linked template.

The last thing required to make the above work is a suitable `.env` file that sets all the relevant `${ENVIRONMENT_VARIABLE}` entries.

Now, this does work, but I am especially curious if there are any security implications. I tried to keep filthy hacks out. The bash script is particularly dumb, but there is [no easy way](https://docs.docker.com/compose/startup-order/) to have a script "wait until you can reach that website".