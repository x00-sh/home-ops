# Home Assistant

Production-ready Home Assistant deployment with 1Password secrets integration.

## Required 1Password Items

Create an item named `home-assistant` in your 1Password "kubernetes" vault with the following fields:

### Required Fields
- `api_token`: Long-lived access token for Home Assistant API
- `secret_key`: Secret key for Home Assistant internal encryption (generate with `openssl rand -hex 32`)

### Optional Location Fields  
- `latitude`: Home latitude coordinate (e.g., "40.7128")
- `longitude`: Home longitude coordinate (e.g., "-74.0060") 
- `elevation`: Home elevation in meters (e.g., "10")

## Features

- **External access**: Available at https://home-assistant.${SECRET_DOMAIN}
- **Secure secrets**: All sensitive data managed via 1Password
- **Optimized storage**: Separate cache and memory-backed temporary storage
- **Non-root security**: Runs with hardened security context
- **Resource efficient**: 10m CPU request with proper limits

## Storage

- **Config**: 5Gi NFS persistent storage for configuration
- **Cache**: 1Gi NFS persistent storage for caches  
- **Logs/TTS/Tmp**: 1Gi memory-backed volumes for performance

## External Dependencies

- **1Password Connect**: Secrets management
- **External Secrets Operator**: Secret synchronization
- **Cloudflare Tunnel**: External access routing