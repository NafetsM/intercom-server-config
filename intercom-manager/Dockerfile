FROM eyevinntechnology/intercom-manager:latest

USER root
RUN sed -i "s/method: 'PUT'/method: 'POST'/" /app/src/smb.ts
USER node