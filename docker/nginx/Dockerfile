# Paneb nginx konfiguratsiooni ja self-signed sertifikaadid image-isse.
# Sertifikaadid genereerib Ansible playbook enne build'i (vt 07-deploy.yml).

FROM nginx:1.27-alpine
RUN mkdir -p /etc/nginx/certs
COPY nginx.conf /etc/nginx/nginx.conf
COPY certs/ /etc/nginx/certs/
EXPOSE 80 443
