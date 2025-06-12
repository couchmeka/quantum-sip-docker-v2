# Quantum-Secure Healthcare Blockchain Network


https://github.com/user-attachments/assets/113ddce1-e744-4be9-b4b7-4c4ee51ac121


This project implements a quantum-resistant healthcare information exchange platform using Hyperledger Fabric and Hedera Hashgraph, with post-quantum cryptography (PQC) integrated throughout all communication layers.

## Architecture Overview

The system connects healthcare organizations (Hospital A and Hospital B) via multiple secure communication channels:

- **Hyperledger Fabric** - Enterprise blockchain for storing verifiable healthcare records
- **Hedera Hashgraph** - For additional consensus and immutable timestamping
- **Asterisk PBX** - For secure voice/video communication with quantum-enhanced SRTP
- **MQTT** - For secure messaging with quantum-resistant encryption
- **Libp2p** - For peer-to-peer networking between organizations

### Post-Quantum Cryptography

The system implements two quantum-resistant algorithms:

- **Falcon-1024** - For digital signatures, replacing ECDSA
- **Kyber-512** - For key encapsulation, replacing RSA/Diffie-Hellman

## Components

### Core Services

| Service | Description | Port(s) |
|---------|-------------|---------|
| `peer0.Hospital_A.example.com` | Fabric peer for Hospital A | 7051 |
| `peer0.Hospital_B.example.com` | Fabric peer for Hospital B | 7061 |
| `orderer.example.com` | Hyperledger Fabric orderer | 7050 |
| `couchdb` | State database for Fabric | 5984 |
| `asterisk` | Quantum-enhanced SIP/VoIP server | 5060-5062, 8088, 8089 |
| `quantum_sip` | SIP service with quantum security | 8000 |
| `quantum_srtp` | Secure Real-time Transport with quantum enhancements | - |
| `quantum_mqtt` | MQTT client with quantum security | - |
| `mqtt` | MQTT broker | 1883, 9883 |
| `wallet-service` | Hedera wallet operations | 3000 |
| `hedera-bridge` | Bridge between Hyperledger and Hedera | - |
| `libp2p-bridge` | P2P networking between organizations | 4001, 8085 |
| `minio` | Object storage | 9000, 9001 |
| `timescaledb` | Time-series database | 5432 |

### Security Components

- **HybridSecuritySystem** - Combines quantum and classical cryptography
- **PostQuantumSessionSecurity** - Session management with quantum resistance
- **QuantumEnhancedSRTP** - Secure Real-time Transport Protocol with quantum key exchange
- **EnhancedEncryption** - Encryption layer with quantum entropy
- **SecureKeyManager** - Manages Falcon and Kyber keys

## Setup and Configuration

### Prerequisites

- Docker and Docker Compose
- Python 3.9+
- Hyperledger Fabric binaries (cryptogen, configtxgen)
- Network access for Hedera integration

### Installation

1. Clone the repository:
   

2. Build the Docker images:
   ```bash
   docker-compose build
   ```

3. Generate cryptographic materials:
   ```bash
   ./init-quantum.sh generate
   ```

4. Start the network:
   ```bash
   docker-compose up -d
   ```

## Configuration Files

- `crypto-config.yaml` - Organization and cryptographic setup
- `docker-compose.yml` - Container configuration
- `network_config.yaml` - Fabric network configuration
- `configs/asterisk/*.conf` - Asterisk configuration files

## Troubleshooting Known Issues

### 1. Orderer Configuration

The orderer requires proper quantum key configuration:

1. Ensure orderer keys are generated:
   ```bash
   # Check if orderer keys exist
   ls -la crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/quantum_keys/
   
   # If missing, run
   python quantum_cryptogen.py generate --config=crypto-config.yaml
   ```

2. Update orderer environment in `docker-compose.yml`:
   ```yaml
   orderer.example.com:
     environment:
       # Add these specific configurations
       - ORDERER_GENERAL_QUANTUM_ENABLED=true
       - ORDERER_GENERAL_QUANTUM_KEYSTORE=/var/hyperledger/orderer/quantum_keys
       - ORDERER_GENERAL_QUANTUM_KEYTYPES=["Falcon","Kyber"]
   ```

### 2. Hospital B Handshake Issues

Possible fixes for Hospital B handshake issues:

1. Check TLS certificates:
   ```bash
   # Verify certificates exist
   ls -la crypto-config/peerOrganizations/Hospital_B.example.com/peers/peer0.Hospital_B.example.com/tls/
   ```

2. Ensure quantum keys are properly generated:
   ```bash
   # Check quantum keys
   ls -la keys/Hospital_B.example.com/
   ```

3. Check network connectivity:
   ```bash
   # From Hospital A container
   docker exec -it peer0.Hospital_A.example.com ping peer0.Hospital_B.example.com
   
   # Test TLS connection
   docker exec -it peer0.Hospital_A.example.com openssl s_client -connect peer0.Hospital_B.example.com:7061
   ```

4. Update SIP configuration in `configs/asterisk/pjsip.conf`:
   ```
   [Hospital_B_endpoint]
   type=endpoint
   transport=transport-tls
   context=from-external
   disallow=all
   allow=ulaw
   allow=alaw
   aors=Hospital_B_endpoint
   auth=Hospital_B_auth
   direct_media=no
   trust_id_inbound=yes
   ```

### 3. Asterisk Not Starting Automatically

To fix Asterisk auto-start issues:

1. Update entrypoint script permissions:
   ```bash
   chmod +x entrypoint_asterisk.sh
   ```

2. Check Asterisk module:
   ```bash
   # Verify module exists
   ls -la asterisk_modules/res_quantum/res_quantum.so
   
   # Ensure module is loaded in config
   grep "res_quantum" configs/asterisk/modules.conf
   ```

3. Update `modules.conf`:
   ```
   [modules]
   autoload=yes
   load => res_quantum.so
   ```

4. Fix directory permissions in Docker startup:
   ```
   # Add to entrypoint_asterisk.sh
   chmod -R 750 /etc/asterisk
   chown -R asterisk:asterisk /etc/asterisk
   ```

### 4. Libp2p Connection Issues

To resolve Libp2p connection issues with Hospital B:

1. Check libp2p configuration in docker-compose.yml:
   ```yaml
   libp2p-bridge:
     environment:
       # Update peer addresses with correct port
       - PEER_ADDRESSES=Hospital_B:7061
       # Ensure TLS is properly configured
       - USE_TLS=true
   ```

2. Check network connectivity:
   ```bash
   # Test connectivity to Hospital B libp2p port
   docker exec -it libp2p-bridge ping peer0.Hospital_B.example.com
   docker exec -it libp2p-bridge nc -zv peer0.Hospital_B.example.com 7061
   ```

3. Check for proper certificate setup:
   ```bash
   # Verify TLS certificates
   ls -la certificates/Hospital_B.example.com/
   ```

## Integration Testing

After resolving configuration issues, test the full integration:

1. Initialize the blockchain with test data:
   ```bash
   docker exec -it cli ./entrypoint.sh
   ```

2. Test SIP connectivity:
   ```bash
   # From Hospital A to Hospital B
   docker exec -it asterisk asterisk -rx "pjsip show endpoint Hospital_B_endpoint"
   ```

3. Test Hedera integration:
   ```bash
   # Submit test transaction
   docker exec -it hedera-bridge python3 -c "from hedera_bridge import HederaFabricBridge; bridge = HederaFabricBridge('Hospital_A'); print(bridge.check_health())"
   ```

## License


## Contact

For assistance, please contact meka.lindquist@mirrorwell.io
