<script>
  import { onMount } from 'svelte';
  import axios from 'axios';

  const API_URL = 'http://localhost:3001/api';

  // Reactive variables
  let chain = [];
  let mempool = [];
  let transactions = [];
  let accounts = {};
  let status = {};
  let selectedBlock = null;
  let selectedTransaction = null;
  let searchAddress = '';
  let addressBalance = null;
  let loading = false;

  // Load data periodically
  let interval;

  onMount(() => {
    loadData();
    interval = setInterval(loadData, 5000);
    return () => clearInterval(interval);
  });

  // Load all data from API
  async function loadData() {
    loading = true;
    try {
      const [chainRes, mempoolRes, txRes, accountsRes, statusRes] = await Promise.all([
        axios.get(`${API_URL}/chain`),
        axios.get(`${API_URL}/mempool`),
        axios.get(`${API_URL}/transactions`),
        axios.get(`${API_URL}/accounts`),
        axios.get(`${API_URL}/status`)
      ]);

      chain = Array.isArray(chainRes?.data) ? chainRes.data : [];
      mempool = Array.isArray(mempoolRes?.data) ? mempoolRes.data : [];
      transactions = Array.isArray(txRes?.data) ? txRes.data : [];
      accounts = accountsRes?.data && typeof accountsRes.data === 'object' ? accountsRes.data : {};
      status = statusRes?.data && typeof statusRes.data === 'object' ? statusRes.data : {};
    } catch (err) {
      console.error('Error loading data:', err);
      chain = [];
      mempool = [];
      transactions = [];
      accounts = {};
      status = {};
    } finally {
      loading = false;
    }
  }

  // Select block or transaction
  function selectBlock(block) {
    selectedBlock = block;
    selectedTransaction = null;
  }

  function selectTransaction(tx) {
    selectedTransaction = tx;
  }

  // Search for an address balance
  async function searchAddressBalance() {
    if (!searchAddress) return;
    try {
      const res = await axios.get(`${API_URL}/balance/${searchAddress}`);
      addressBalance = res.data;
    } catch (err) {
      console.error('Error fetching balance:', err);
      addressBalance = { error: 'Address not found' };
    }
  }

  // Utility functions
  function formatHash(hash) {
    return hash ? `${hash.slice(0, 16)}...${hash.slice(-16)}` : 'N/A';
  }

  function formatTimestamp(timestamp) {
    return timestamp ? new Date(timestamp).toLocaleString() : 'N/A';
  }
</script>

<svelte:window on:keydown={(e) => e.key === 'Escape' && (selectedBlock = selectedTransaction = null)} />

<div class="app">
  <header class="topbar">
    <div class="topbar-inner">
      <div class="logo">
        <img src="/vite.svg" alt="FlowChain" />
        <div class="title">
          <div class="brand">FlowChain</div>
          <div class="subtitle">Explorer</div>
        </div>
      </div>

      <div class="search-bar">
        <input type="text" placeholder="Search by address / tx hash / block #" bind:value={searchAddress} on:keydown={(e) => e.key === 'Enter' && searchAddressBalance()} />
        <button on:click={searchAddressBalance}>{loading ? 'Searching...' : 'Search'}</button>
      </div>

      <div class="top-stats">
        <div class="stat"><div class="num">{status.chainLength ?? 0}</div><div class="lbl">Blocks</div></div>
        <div class="stat"><div class="num">{status.mempoolSize ?? 0}</div><div class="lbl">Pending</div></div>
        <div class="stat"><div class="num {status.isValid ? 'valid' : 'invalid'}">{status.isValid ? '✓' : '✗'}</div><div class="lbl">Chain</div></div>
        <button class="refresh-btn" on:click={loadData} disabled={loading}>{loading ? 'Loading...' : 'Refresh'}</button>
      </div>
    </div>
  </header>

  <div class="main-content etherscan-style">
    <aside class="sidebar">
      <div class="blocks-section">
        <h3>Latest Blocks</h3>
        <div class="blocks-list">
          {#each chain.slice().reverse().slice(0, 20) as block, index}
            <div class="block-item compact" on:click={() => selectBlock(block)}>
              <div class="left">
                <div class="block-number">#{chain.length - index - 1}</div>
                <div class="block-hash">{formatHash(block?.hash)}</div>
              </div>
              <div class="right">
                <div class="txs">{block?.transactions?.length ?? 0} tx</div>
                <div class="time">{formatTimestamp(block?.timestamp)}</div>
              </div>
            </div>
          {/each}
        </div>
      </div>
    </aside>

    <div class="content">
      {#if selectedBlock}
        <div class="detail-view">
          <button class="close-btn" on:click={() => selectedBlock = null}>✕</button>
          <h2>Block #{chain.indexOf(selectedBlock)}</h2>
          <div class="detail-grid">
            <div class="detail-item">
              <label>Hash:</label>
              <code>{selectedBlock?.hash ?? 'N/A'}</code>
            </div>
            <div class="detail-item">
              <label>Previous Hash:</label>
              <code>{selectedBlock?.previousHash ?? 'N/A'}</code>
            </div>
            <div class="detail-item">
              <label>Timestamp:</label>
              <span>{formatTimestamp(selectedBlock?.timestamp)}</span>
            </div>
            <div class="detail-item">
              <label>Nonce:</label>
              <span>{selectedBlock?.nonce ?? 'N/A'}</span>
            </div>
            <div class="detail-item">
              <label>Transactions:</label>
              <span>{selectedBlock?.transactions?.length ?? 0}</span>
            </div>
          </div>

          <h3>Transactions</h3>
          <div class="transactions-list">
            {#each selectedBlock?.transactions ?? [] as tx}
              <div
                class="transaction-item"
                class:selected={selectedTransaction === tx}
                on:click={() => selectTransaction(tx)}
              >
                <div class="tx-header">
                  <span class="tx-hash">{formatHash(tx?.hash)}</span>
                  <span class="tx-amount">{tx?.amount ?? 0} FLOW</span>
                </div>
                <div class="tx-details">
                  <div>From: {tx?.from || 'Mining Reward'}</div>
                  <div>To: {tx?.to ?? 'N/A'}</div>
                  {#if tx?.gasFee}
                    <div class="tx-gas">Gas: {tx.gasFee.toFixed(6)} Gas Tokens</div>
                  {/if}
                </div>
              </div>
            {/each}
          </div>
        </div>

      {:else if selectedTransaction}
        <div class="detail-view">
          <button class="close-btn" on:click={() => selectedTransaction = null}>✕</button>
          <h2>Transaction Details</h2>
          <div class="detail-grid">
            <div class="detail-item">
              <label>Hash:</label>
              <code>{selectedTransaction?.hash ?? 'N/A'}</code>
            </div>
            <div class="detail-item">
              <label>From:</label>
              <code>{selectedTransaction?.from || 'Mining Reward'}</code>
            </div>
            <div class="detail-item">
              <label>To:</label>
              <code>{selectedTransaction?.to ?? 'N/A'}</code>
            </div>
            <div class="detail-item">
              <label>Amount:</label>
              <span class="amount">{selectedTransaction?.amount ?? 0} FLOW</span>
            </div>
            {#if selectedTransaction?.gasFee}
              <div class="detail-item">
                <label>Gas Fee:</label>
                <span class="gas-fee">{selectedTransaction.gasFee.toFixed(6)} Gas Tokens</span>
              </div>
              <div class="detail-item">
                <label>Gas Used:</label>
                <span>{selectedTransaction?.gasUsed ?? 'N/A'}</span>
              </div>
              <div class="detail-item">
                <label>Gas Price:</label>
                <span>{selectedTransaction?.gasPrice ?? 'N/A'}</span>
              </div>
            {/if}
            <div class="detail-item">
              <label>Timestamp:</label>
              <span>{formatTimestamp(selectedTransaction?.timestamp)}</span>
            </div>
            {#if selectedTransaction?.data}
              <div class="detail-item">
                <label>Data:</label>
                <pre>{JSON.stringify(selectedTransaction.data, null, 2)}</pre>
              </div>
            {/if}
          </div>
        </div>

      {:else}
        <div class="overview">
          <!-- Mempool -->
          <div class="section">
            <h2>Mempool ({mempool?.length ?? 0})</h2>
            {#if !mempool?.length}
              <p class="empty">No pending transactions</p>
            {:else}
              <div class="transactions-list">
                {#each mempool ?? [] as tx}
                  <div class="transaction-item" on:click={() => selectTransaction(tx)}>
                    <div class="tx-header">
                      <span class="tx-hash">{formatHash(tx?.hash)}</span>
                      <span class="tx-amount">{tx?.amount ?? 0} FLOW</span>
                    </div>
                    <div class="tx-details">
                      <div>From: {tx?.from || 'Mining Reward'}</div>
                      <div>To: {tx?.to ?? 'N/A'}</div>
                      {#if tx?.gasFee}
                        <div class="tx-gas">Gas: {tx.gasFee.toFixed(6)} Gas Tokens</div>
                      {/if}
                    </div>
                  </div>
                {/each}
              </div>
            {/if}
          </div>

          <!-- Recent Transactions -->
          <div class="section">
            <h2>Recent Transactions</h2>
            <div class="transactions-list">
              {#each (transactions ?? []).slice().reverse().slice(0, 20) as tx}
                <div class="transaction-item" on:click={() => selectTransaction(tx)}>
                  <div class="tx-header">
                    <span class="tx-hash">{formatHash(tx?.hash)}</span>
                    <span class="tx-amount">{tx?.amount ?? 0} FLOW</span>
                  </div>
                  <div class="tx-details">
                    <div>From: {tx?.from || 'Mining Reward'}</div>
                    <div>To: {tx?.to ?? 'N/A'}</div>
                    {#if tx?.gasFee}
                      <div class="tx-gas">Gas: {tx.gasFee.toFixed(6)} Gas Tokens</div>
                    {/if}
                    <div class="tx-time">{formatTimestamp(tx?.timestamp)}</div>
                  </div>
                </div>
              {/each}
            </div>
          </div>
        </div>
      {/if}
    </div>
  </div>
</div>


<style>
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: #f5f5f5;
  }

  .app {
    min-height: 100vh;
  }

  /* Topbar styled similar to Etherscan */
  .topbar {
    background: #ffffff;
    border-bottom: 1px solid #e6e6e6;
    padding: 12px 24px;
    position: sticky;
    top: 0;
    z-index: 20;
  }

  .topbar-inner {
    max-width: 1400px;
    margin: 0 auto;
    display: flex;
    gap: 20px;
    align-items: center;
  }

  .logo {
    display: flex;
    gap: 12px;
    align-items: center;
  }

  .logo img { width: 36px; height: 36px; }
  .logo .brand { font-weight: 700; font-size: 18px; color: #111827; }
  .logo .subtitle { font-size: 12px; color: #6b7280; }

  .search-bar { flex: 1; display: flex; gap: 8px; }
  .search-bar input { flex: 1; padding: 10px 12px; border: 1px solid #e5e7eb; border-radius: 6px; }
  .search-bar button { padding: 10px 14px; background: #3b82f6; color: white; border: none; border-radius: 6px; cursor: pointer; }

  .top-stats { display: flex; gap: 16px; align-items: center; }
  .top-stats .stat { text-align: center; min-width: 68px; }
  .top-stats .num { font-weight: 700; color: #111827; }
  .top-stats .lbl { font-size: 12px; color: #6b7280; }
  .top-stats .num.valid { color: #16a34a; }
  .top-stats .num.invalid { color: #ef4444; }

  .refresh-btn { padding: 8px 12px; border-radius: 6px; background: #f3f4f6; border: 1px solid #e5e7eb; cursor: pointer; }

  .main-content.etherscan-style { display: flex; max-width: 1400px; margin: 20px auto; padding: 0 24px; gap: 20px; }

  .sidebar {
    width: 350px;
    flex-shrink: 0;
  }

  .search-section,
  .blocks-section {
    background: white;
    padding: 20px;
    border-radius: 10px;
    margin-bottom: 20px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  }

  .search-section h3,
  .blocks-section h3 {
    margin-bottom: 15px;
    color: #333;
  }

  .search-section input {
    width: 100%;
    padding: 10px;
    border: 2px solid #e0e0e0;
    border-radius: 6px;
    margin-bottom: 10px;
    font-size: 14px;
  }

  .search-section button {
    width: 100%;
    padding: 10px;
    background: #667eea;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
  }

  .search-section button:hover {
    background: #5568d3;
  }

  .balance-result {
    margin-top: 15px;
    padding: 10px;
    background: #f0f0f0;
    border-radius: 6px;
  }

  .balance-result .error {
    color: #dc3545;
  }

  .blocks-list {
    max-height: 600px;
    overflow-y: auto;
  }

  .block-item {
    padding: 15px;
    background: #f8f9fa;
    border-radius: 8px;
    margin-bottom: 10px;
    cursor: pointer;
    border: 2px solid transparent;
    transition: all 0.2s;
  }

  .block-item:hover {
    border-color: #667eea;
    background: #f0f0f5;
  }

  .block-item.selected {
    border-color: #667eea;
    background: #e8e9ff;
  }

  .block-number {
    font-weight: 600;
    color: #667eea;
    margin-bottom: 5px;
  }

  .block-hash {
    font-family: 'Courier New', monospace;
    font-size: 12px;
    color: #666;
    margin-bottom: 5px;
  }

  .block-txs {
    font-size: 12px;
    color: #999;
  }

  .content {
    flex: 1;
    min-width: 0;
  }

  .overview {
    display: flex;
    flex-direction: column;
    gap: 20px;
  }

  .section {
    background: white;
    padding: 25px;
    border-radius: 10px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  }

  .section h2 {
    margin-bottom: 20px;
    color: #333;
  }

  .empty {
    color: #999;
    text-align: center;
    padding: 40px;
  }

  .detail-view {
    background: white;
    padding: 30px;
    border-radius: 10px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    position: relative;
  }

  .close-btn {
    position: absolute;
    top: 20px;
    right: 20px;
    background: #f0f0f0;
    border: none;
    width: 30px;
    height: 30px;
    border-radius: 50%;
    cursor: pointer;
    font-size: 18px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .close-btn:hover {
    background: #e0e0e0;
  }

  .detail-view h2 {
    margin-bottom: 20px;
    color: #333;
  }

  .detail-view h3 {
    margin: 30px 0 15px 0;
    color: #555;
  }

  .detail-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 20px;
    margin-bottom: 30px;
  }

  .detail-item {
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .detail-item label {
    font-weight: 600;
    color: #666;
    font-size: 14px;
  }

  .detail-item code {
    background: #f8f9fa;
    padding: 8px;
    border-radius: 4px;
    font-family: 'Courier New', monospace;
    font-size: 13px;
    word-break: break-all;
  }

  .detail-item .amount {
    font-size: 18px;
    font-weight: 600;
    color: #28a745;
  }

  .detail-item .gas-fee {
    font-size: 16px;
    font-weight: 600;
    color: #ff9800;
  }

  .detail-item pre {
    background: #f8f9fa;
    padding: 10px;
    border-radius: 4px;
    overflow-x: auto;
    font-size: 12px;
  }

  .transactions-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .transaction-item {
    padding: 15px;
    background: #f8f9fa;
    border-radius: 8px;
    cursor: pointer;
    border: 2px solid transparent;
    transition: all 0.2s;
  }

  .transaction-item:hover {
    border-color: #667eea;
    background: #f0f0f5;
  }

  .transaction-item.selected {
    border-color: #667eea;
    background: #e8e9ff;
  }

  .tx-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
  }

  .tx-hash {
    font-family: 'Courier New', monospace;
    font-size: 13px;
    color: #666;
  }

  .tx-amount {
    font-weight: 600;
    color: #28a745;
  }

  .tx-details {
    font-size: 13px;
    color: #666;
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .tx-time {
    font-size: 11px;
    color: #999;
    margin-top: 5px;
  }

  .tx-gas {
    font-size: 12px;
    color: #ff9800;
    font-weight: 600;
  }

  /* Sidebar tweaks for etherscan-style */
  .sidebar {
    width: 360px;
    flex-shrink: 0;
  }

  .blocks-section h3 { margin-bottom: 12px; }

  .block-item.compact {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 12px;
    background: #ffffff;
    border-radius: 8px;
    border: 1px solid #e6e6e6;
    margin-bottom: 10px;
  }

  .block-item.compact .left { display: flex; flex-direction: column; gap: 4px; }
  .block-item.compact .right { text-align: right; font-size: 12px; color: #6b7280; }
  .block-item.compact .block-number { font-weight: 700; color: #111827; }
  .block-item.compact .block-hash { font-family: monospace; color: #6b7280; font-size: 12px; }

  /* Content column: widen and use cards */
  .content { flex: 1; min-width: 0; }

  /* Make detail-view larger and Etherscan-like */
  .detail-view { padding: 24px; }
</style>

