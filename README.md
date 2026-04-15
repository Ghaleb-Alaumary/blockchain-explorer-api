# blockchain-explorer-api
EST API wrapper for multiple blockchain explorers and address monitoring

# Blockchain Explorer API

Unified REST API interface for querying multiple blockchain explorers. Supports address balance checks, transaction history, and real-time monitoring across Bitcoin, Ethereum, TRON, and privacy networks.

## Supported Blockchain Networks

| Network | Explorer | API Endpoint | Status |
|---------|----------|--------------|--------|
| Bitcoin | Blockstream, Blockchain.com | esplora, blockchain.info | ✅ Stable |
| Ethereum | Etherscan | api.etherscan.io | ✅ Stable |
| TRON | Tronscan | api.tronscan.org | ✅ Stable |
| Monero | XMRChain | xmrchain.net | ⚠️ Beta |
| Binance Smart Chain | BSCScan | api.bscscan.com | ✅ Stable |

## Installation

```bash
npm install blockchain-explorer-api
# or
yarn add blockchain-explorer-api

Quick Start

const BlockchainAPI = require('blockchain-explorer-api');

// Initialize with API keys
const api = new BlockchainAPI({
    etherscan: 'YOUR_ETHERSCAN_API_KEY',
    bscscan: 'YOUR_BSCSCAN_API_KEY',
    tronscan: 'YOUR_TRONSCAN_API_KEY'
});

// Get Ethereum balance
const ethBalance = await api.ethereum.getBalance('0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb7');
console.log(`ETH Balance: ${ethBalance} ETH`);

// Get Bitcoin address info
const btcInfo = await api.bitcoin.getAddress('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa');
console.log(`BTC Balance: ${btcInfo.balance} BTC`);
console.log(`Total Transactions: ${btcInfo.txCount}`);

// Get TRON account details
const tronAccount = await api.tron.getAccount('TJCnKsPa7y5okkXvQidzrVtDZk9dQrFg2T');
console.log(`TRX Balance: ${tronAccount.balance} TRX`);

Advanced Usage
Multi-Address Monitoring

const monitor = api.createMonitor({
    networks: ['ethereum', 'bitcoin', 'tron'],
    addresses: [
        '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb7',
        '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
        'TJCnKsPa7y5okkXvQidzrVtDZk9dQrFg2T'
    ],
    pollingInterval: 30000 // 30 seconds
});

monitor.on('transaction', (tx) => {
    console.log(`New transaction detected on ${tx.network}:`);
    console.log(`  Hash: ${tx.hash}`);
    console.log(`  Amount: ${tx.value} ${tx.symbol}`);
    console.log(`  From: ${tx.from}`);
    console.log(`  To: ${tx.to}`);
});

monitor.start();

Transaction Tracing

// Trace ETH transaction
const txTrace = await api.ethereum.traceTransaction(
    '0x5c504ed432cb51138bcf09aa5e8a410dd4a1e204ef84bfed1be16dfba1b22060'
);

// Identify potential mixing services
const riskScore = await api.analyzeRisk(txTrace);
console.log(`Risk Score: ${riskScore}/100`);

API Reference
Bitcoin Methods
getAddress(address) - Get address balance and stats

getTransactions(address, limit) - Get transaction history

getTransaction(txid) - Get transaction details

getUTXOs(address) - Get unspent transaction outputs

Ethereum Methods
getBalance(address) - Get ETH balance

getTokenBalance(address, contract) - Get ERC20 token balance

getTransactions(address, limit) - Get transaction history

getTransaction(txid) - Get transaction details

getContractABI(address) - Get verified contract ABI

TRON Methods
getAccount(address) - Get account details

getTransactions(address, limit) - Get transaction history

getTRC20Balance(address, contract) - Get TRC20 token balance

Configuration

{
  "networks": {
    "ethereum": {
      "apiKey": "YOUR_KEY",
      "endpoint": "https://api.etherscan.io/api"
    },
    "bitcoin": {
      "endpoint": "https://blockstream.info/api"
    },
    "tron": {
      "apiKey": "YOUR_KEY",
      "endpoint": "https://api.tronscan.org/api"
    }
  },
  "monitoring": {
    "defaultPollingInterval": 30000,
    "maxAddressesPerMonitor": 100,
    "webhookUrl": "https://your-server.com/webhook"
  }
}

License
Proprietary - Internal Use Only


### `index.js`
```javascript
/**
 * Blockchain Explorer API Wrapper
 * Version: 2.4.1
 * Author: G. Alaumary
 * Date: 2021-09-08
 */

const axios = require('axios');
const EventEmitter = require('events');

class BlockchainAPI {
    constructor(config = {}) {
        this.config = {
            etherscan: config.etherscan || process.env.ETHERSCAN_API_KEY,
            bscscan: config.bscscan || process.env.BSCSCAN_API_KEY,
            tronscan: config.tronscan || process.env.TRONSCAN_API_KEY,
            ...config
        };
        
        // Initialize network handlers
        this.ethereum = new EthereumHandler(this.config);
        this.bitcoin = new BitcoinHandler(this.config);
        this.tron = new TronHandler(this.config);
        this.bsc = new BSCHandler(this.config);
        
        // Rate limiting
        this.rateLimits = {
            etherscan: { max: 5, window: 1000, requests: [] },
            blockchain_info: { max: 10, window: 60000, requests: [] },
            tronscan: { max: 20, window: 1000, requests: [] }
        };
    }
    
    createMonitor(options = {}) {
        return new AddressMonitor(this, options);
    }
    
    async analyzeRisk(transactionData) {
        // Simplified risk scoring
        let score = 0;
        
        // Check for known mixer addresses
        const knownMixers = [
            '0x722122dF12D4e14e13Ac3B6895a86e84145b6967', // Tornado Cash
            '0x12D66f87A04A9E220743712cE6d9bB1B5616B8Fc', // Tornado Cash
            '1NDyJtNTjmwk5xPNhjgAMu4HDHigtobu1s' // Bitcoin Fog (historical)
        ];
        
        if (transactionData.to && knownMixers.includes(transactionData.to)) {
            score += 50;
        }
        
        // Check value patterns
        if (transactionData.value && transactionData.value > 100000) {
            score += 10;
        }
        
        return Math.min(score, 100);
    }
}

class EthereumHandler {
    constructor(config) {
        this.apiKey = config.etherscan;
        this.baseURL = 'https://api.etherscan.io/api';
    }
    
    async getBalance(address) {
        const response = await axios.get(this.baseURL, {
            params: {
                module: 'account',
                action: 'balance',
                address: address,
                tag: 'latest',
                apikey: this.apiKey
            }
        });
        
        if (response.data.status === '1') {
            return parseFloat(response.data.result) / 1e18;
        }
        throw new Error(`Etherscan API error: ${response.data.message}`);
    }
    
    async getTokenBalance(address, contractAddress) {
        const response = await axios.get(this.baseURL, {
            params: {
                module: 'account',
                action: 'tokenbalance',
                contractaddress: contractAddress,
                address: address,
                tag: 'latest',
                apikey: this.apiKey
            }
        });
        
        return response.data.result;
    }
    
    async getTransactions(address, limit = 10) {
        const response = await axios.get(this.baseURL, {
            params: {
                module: 'account',
                action: 'txlist',
                address: address,
                startblock: 0,
                endblock: 99999999,
                page: 1,
                offset: limit,
                sort: 'desc',
                apikey: this.apiKey
            }
        });
        
        return response.data.result || [];
    }
    
    async traceTransaction(txHash) {
        const response = await axios.get(this.baseURL, {
            params: {
                module: 'account',
                action: 'txlistinternal',
                txhash: txHash,
                apikey: this.apiKey
            }
        });
        
        return {
            hash: txHash,
            internalCalls: response.data.result || []
        };
    }
}

class BitcoinHandler {
    constructor(config) {
        this.baseURL = 'https://blockstream.info/api';
    }
    
    async getAddress(address) {
        const response = await axios.get(`${this.baseURL}/address/${address}`);
        
        const chainStats = response.data.chain_stats;
        const mempoolStats = response.data.mempool_stats;
        
        return {
            address: address,
            balance: (chainStats.funded_txo_sum - chainStats.spent_txo_sum) / 1e8,
            txCount: chainStats.tx_count,
            fundedSum: chainStats.funded_txo_sum / 1e8,
            spentSum: chainStats.spent_txo_sum / 1e8
        };
    }
    
    async getTransactions(address, limit = 10) {
        const response = await axios.get(`${this.baseURL}/address/${address}/txs`);
        return response.data.slice(0, limit);
    }
    
    async getUTXOs(address) {
        const response = await axios.get(`${this.baseURL}/address/${address}/utxo`);
        return response.data.map(utxo => ({
            txid: utxo.txid,
            vout: utxo.vout,
            value: utxo.value / 1e8,
            status: utxo.status
        }));
    }
}

class TronHandler {
    constructor(config) {
        this.apiKey = config.tronscan;
        this.baseURL = 'https://apilist.tronscan.org/api';
    }
    
    async getAccount(address) {
        const response = await axios.get(`${this.baseURL}/account`, {
            params: { address: address }
        });
        
        const data = response.data;
        
        return {
            address: address,
            balance: data.balance / 1e6,
            bandwidth: data.bandwidth,
            energy: data.energy,
            totalTxCount: data.totalTransactionCount
        };
    }
    
    async getTransactions(address, limit = 10) {
        const response = await axios.get(`${this.baseURL}/transaction`, {
            params: {
                address: address,
                limit: limit,
                sort: '-timestamp'
            }
        });
        
        return response.data.data || [];
    }
    
    async getTRC20Balance(address, contractAddress) {
        const response = await axios.get(`${this.baseURL}/account/token_asset`, {
            params: { address: address }
        });
        
        const tokens = response.data.data || [];
        const token = tokens.find(t => t.tokenId === contractAddress);
        
        return token ? parseFloat(token.balance) / Math.pow(10, token.tokenDecimal) : 0;
    }
}

class BSCHandler extends EthereumHandler {
    constructor(config) {
        super(config);
        this.baseURL = 'https://api.bscscan.com/api';
    }
}

class AddressMonitor extends EventEmitter {
    constructor(api, options = {}) {
        super();
        this.api = api;
        this.networks = options.networks || ['ethereum'];
        this.addresses = options.addresses || [];
        this.pollingInterval = options.pollingInterval || 30000;
        this._running = false;
        this._lastChecked = {};
        this._knownTransactions = new Set();
    }
    
    start() {
        this._running = true;
        this._poll();
        console.log(`Monitor started for ${this.addresses.length} addresses on ${this.networks.join(', ')}`);
    }
    
    stop() {
        this._running = false;
        console.log('Monitor stopped');
    }
    
    async _poll() {
        while (this._running) {
            for (const address of this.addresses) {
                for (const network of this.networks) {
                    await this._checkAddress(network, address);
                }
            }
            await this._sleep(this.pollingInterval);
        }
    }
    
    async _checkAddress(network, address) {
        try {
            let transactions = [];
            
            if (network === 'ethereum') {
                transactions = await this.api.ethereum.getTransactions(address, 5);
            } else if (network === 'bitcoin') {
                transactions = await this.api.bitcoin.getTransactions(address, 5);
            } else if (network === 'tron') {
                transactions = await this.api.tron.getTransactions(address, 5);
            }
            
            for (const tx of transactions) {
                const txId = tx.hash || tx.txid;
                if (!this._knownTransactions.has(txId)) {
                    this._knownTransactions.add(txId);
                    this.emit('transaction', {
                        network: network,
                        hash: txId,
                        from: tx.from || tx.vin?.[0]?.addresses?.[0],
                        to: tx.to || tx.vout?.[0]?.addresses?.[0],
                        value: this._extractValue(network, tx),
                        symbol: network === 'ethereum' ? 'ETH' : 
                                network === 'bitcoin' ? 'BTC' : 'TRX',
                        timestamp: tx.time || tx.timestamp
                    });
                }
            }
        } catch (error) {
            this.emit('error', { network, address, error: error.message });
        }
    }
    
    _extractValue(network, tx) {
        if (network === 'ethereum') {
            return parseFloat(tx.value) / 1e18;
        } else if (network === 'bitcoin') {
            const value = tx.vout?.reduce((sum, out) => sum + out.value, 0) || 0;
            return value / 1e8;
        } else if (network === 'tron') {
            return tx.amount / 1e6;
        }
        return 0;
    }
    
    _sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Known high-risk addresses (for monitoring)
const WATCHLIST = {
    mixers: [
        '0x722122dF12D4e14e13Ac3B6895a86e84145b6967', // Tornado Cash ETH
        '0x12D66f87A04A9E220743712cE6d9bB1B5616B8Fc', // Tornado Cash ETH
        '0x47CE0C6eD5B0Ce3d3A51fdb1C52DC66a7c3c2936', // Tornado Cash ETH
    ],
    exchanges_hk: [
        '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb7', // OTC desk HK
        '0x5C985E89dDe482eFE97ea9a1950aD149Eb73829B', // Exchange wallet
    ]
};

module.exports = BlockchainAPI;
module.exports.WATCHLIST = WATCHLIST;


