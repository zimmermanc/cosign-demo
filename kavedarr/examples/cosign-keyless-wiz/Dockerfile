FROM nginx:1.27-alpine

# Add demo content
RUN echo '<html><body><h1>Cosign Demo - Signed Container</h1><p>This container was signed with Cosign!</p><p>Current time: <span id="time"></span></p><script>document.getElementById("time").innerHTML = new Date().toLocaleString();</script></body></html>' > /usr/share/nginx/html/index.html

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

EXPOSE 80