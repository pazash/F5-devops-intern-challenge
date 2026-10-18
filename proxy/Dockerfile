# Proxy image inherited from the previous team.
FROM busybox:1.36.1-uclibc AS busybox

FROM cgr.dev/chainguard/nginx:latest

COPY --from=busybox /bin/wget /usr/bin/wget

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 8080

