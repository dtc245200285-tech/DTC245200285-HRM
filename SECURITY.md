# Security notes

This is an educational lab project, not a production-ready HR system. It does not implement authentication or authorization. Do not expose it to the public Internet or enter real personal data. Set strong unique secrets in `.env`, restrict host firewall access, use TLS in front of Nginx, keep images patched, and configure backups before any real deployment. Promtail's Docker socket mount can expose host-level container metadata and should be treated as privileged access.
