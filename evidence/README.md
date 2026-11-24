Week 13 Reflection

Kubernetes automatically created replacement pods when I deleted them to maintain the desired number of replicas.

The liveness probe checks whether the application is running, while the readiness probe checks whether the application is ready to receive traffic and can connect to PostgreSQL.

The database ticket survived pod deletion because PostgreSQL stores its data on a persistent volume rather than inside the pod itself.
