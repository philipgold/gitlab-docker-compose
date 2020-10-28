FROM gitlab/gitlab-runner

COPY ./runner_registration.sh /

RUN chmod +x /runner_registration.sh

ENTRYPOINT [ "/bin/bash", "/runner_registration.sh" ]
