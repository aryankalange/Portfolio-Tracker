/** @jsxImportSource https://esm.sh/react@18.2.0 */
import React, { useState, useEffect } from "https://esm.sh/react@18.2.0";
import { createRoot } from "https://esm.sh/react-dom@18.2.0/client";

// Finnhub API for stock prices (free tier)
const FINNHUB_API_KEY = 'cn3b6l9r01qjqnqvg670cn3b6l9r01qjqnqvg67g';

function App() {
  const [assets, setAssets] = useState([]);
  const [editingAsset, setEditingAsset] = useState(null);
  const [newAsset, setNewAsset] = useState({ symbol: '', shares: 1, purchasePrice: 0 });
  const [portfolioMetrics, setPortfolioMetrics] = useState({
    totalValue: 0,
    totalCost: 0,
    gainLoss: 0,
    gainLossPercentage: 0
  });

  useEffect(() => {
    fetchPortfolio();
  }, []);

  const fetchPortfolio = async () => {
    const response = await fetch('/portfolio');
    const data = await response.json();
    setAssets(data);
    await calculatePortfolioMetrics(data);
  };

  const calculatePortfolioMetrics = async (portfolioAssets) => {
    const prices = await Promise.all(
      portfolioAssets.map(asset => 
        fetch(`https://finnhub.io/api/v1/quote?symbol=${asset.symbol}&token=${FINNHUB_API_KEY}`)
          .then(res => res.json())
          .then(data => ({ 
            symbol: asset.symbol, 
            currentPrice: data.c || asset.purchasePrice,
            changePercent: data.dp || 0
          }))
      )
    );

    const totalCost = portfolioAssets.reduce((total, asset) => 
      total + (asset.shares * asset.purchasePrice), 0);

    const totalValue = portfolioAssets.reduce((total, asset) => {
      const price = prices.find(p => p.symbol === asset.symbol)?.currentPrice || asset.purchasePrice;
      return total + (asset.shares * price);
    }, 0);

    const gainLoss = totalValue - totalCost;
    const gainLossPercentage = (gainLoss / totalCost) * 100;

    setPortfolioMetrics({
      totalValue,
      totalCost,
      gainLoss,
      gainLossPercentage
    });
  };

  const addAsset = async (e) => {
    e.preventDefault();
    const response = await fetch('/portfolio', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(newAsset)
    });
    
    if (response.ok) {
      fetchPortfolio();
      setNewAsset({ symbol: '', shares: 1, purchasePrice: 0 });
    }
  };

  const updateAsset = async (asset) => {
    const response = await fetch(`/portfolio/${asset.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(asset)
    });
    
    if (response.ok) {
      fetchPortfolio();
      setEditingAsset(null);
    }
  };

  const deleteAsset = async (assetId) => {
    const response = await fetch(`/portfolio/${assetId}`, {
      method: 'DELETE'
    });
    
    if (response.ok) {
      fetchPortfolio();
    }
  };

  return (
    <div style={{ 
      maxWidth: '800px', 
      margin: 'auto', 
      padding: '20px', 
      fontFamily: 'Arial, sans-serif' 
    }}>
      <h1>📈 Portfolio Tracker</h1>
      
      <div style={{ 
        display: 'flex', 
        justifyContent: 'space-between', 
        marginBottom: '20px',
        padding: '15px',
        backgroundColor: '#f4f4f4',
        borderRadius: '8px'
      }}>
        <div>
          <h3>Total Portfolio Value</h3>
          <p style={{ 
            color: portfolioMetrics.gainLoss >= 0 ? 'green' : 'red',
            fontSize: '1.5em',
            fontWeight: 'bold'
          }}>
            ${portfolioMetrics.totalValue.toFixed(2)}
          </p>
        </div>
        <div>
          <h3>Total Gain/Loss</h3>
          <p style={{ 
            color: portfolioMetrics.gainLoss >= 0 ? 'green' : 'red',
            fontSize: '1.2em'
          }}>
            ${portfolioMetrics.gainLoss.toFixed(2)} 
            ({portfolioMetrics.gainLossPercentage.toFixed(2)}%)
          </p>
        </div>
      </div>

      <form 
        onSubmit={addAsset} 
        style={{ 
          display: 'flex', 
          gap: '10px', 
          marginBottom: '20px' 
        }}
      >
        <input 
          type="text" 
          placeholder="Stock Symbol" 
          value={newAsset.symbol}
          onChange={(e) => setNewAsset({...newAsset, symbol: e.target.value.toUpperCase()})}
          required 
          style={{ flex: 1, padding: '10px' }}
        />
        <input 
          type="number" 
          placeholder="Shares" 
          value={newAsset.shares}
          onChange={(e) => setNewAsset({...newAsset, shares: Number(e.target.value)})}
          required 
          style={{ width: '100px', padding: '10px' }}
        />
        <input 
          type="number" 
          placeholder="Purchase Price" 
          value={newAsset.purchasePrice}
          onChange={(e) => setNewAsset({...newAsset, purchasePrice: Number(e.target.value)})}
          required 
          style={{ width: '150px', padding: '10px' }}
        />
        <button 
          type="submit" 
          style={{ 
            padding: '10px 15px', 
            backgroundColor: '#4CAF50', 
            color: 'white', 
            border: 'none', 
            borderRadius: '5px' 
          }}
        >
          Add Asset
        </button>
      </form>

      <table style={{ width: '100%', borderCollapse: 'collapse' }}>
        <thead>
          <tr style={{ backgroundColor: '#f2f2f2' }}>
            <th style={tableHeaderStyle}>Symbol</th>
            <th style={tableHeaderStyle}>Shares</th>
            <th style={tableHeaderStyle}>Purchase Price</th>
            <th style={tableHeaderStyle}>Current Value</th>
            <th style={tableHeaderStyle}>Actions</th>
          </tr>
        </thead>
        <tbody>
          {assets.map((asset) => (
            <tr key={asset.id} style={{ borderBottom: '1px solid #ddd' }}>
              {editingAsset?.id === asset.id ? (
                <>
                  <td><input 
                    value={editingAsset.symbol} 
                    onChange={(e) => setEditingAsset({...editingAsset, symbol: e.target.value.toUpperCase()})}
                  /></td>
                  <td><input 
                    type="number" 
                    value={editingAsset.shares} 
                    onChange={(e) => setEditingAsset({...editingAsset, shares: Number(e.target.value)})}
                  /></td>
                  <td><input 
                    type="number" 
                    value={editingAsset.purchasePrice} 
                    onChange={(e) => setEditingAsset({...editingAsset, purchasePrice: Number(e.target.value)})}
                  /></td>
                  <td></td>
                  <td>
                    <button onClick={() => updateAsset(editingAsset)}>Save</button>
                    <button onClick={() => setEditingAsset(null)}>Cancel</button>
                  </td>
                </>
              ) : (
                <>
                  <td>{asset.symbol}</td>
                  <td>{asset.shares}</td>
                  <td>${asset.purchasePrice.toFixed(2)}</td>
                  <td>${(asset.shares * asset.purchasePrice).toFixed(2)}</td>
                  <td>
                    <button 
                      onClick={() => setEditingAsset(asset)}
                      style={{ marginRight: '5px' }}
                    >
                      Edit
                    </button>
                    <button 
                      onClick={() => deleteAsset(asset.id)}
                      style={{ backgroundColor: '#f44336', color: 'white' }}
                    >
                      Delete
                    </button>
                  </td>
                </>
              )}
            </tr>
          ))}
        </tbody>
      </table>

      <a 
        href={import.meta.url.replace("esm.town", "val.town")} 
        target="_top" 
        style={{ color: '#888', textDecoration: 'none', fontSize: '0.8em' }}
      >
        View Source
      </a>
    </div>
  );
}

const tableHeaderStyle = {
  padding: '12px',
  textAlign: 'left',
  borderBottom: '1px solid #ddd'
};

function client() {
  createRoot(document.getElementById("root")).render(<App />);
}
if (typeof document !== "undefined") { client(); }

export default async function server(request: Request): Promise<Response> {
  const { sqlite } = await import("https://esm.town/v/stevekrouse/sqlite");
  const KEY = new URL(import.meta.url).pathname.split("/").at(-1);
  const SCHEMA_VERSION = 2;

  await sqlite.execute(`
    CREATE TABLE IF NOT EXISTS ${KEY}_portfolio_${SCHEMA_VERSION} (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      symbol TEXT NOT NULL,
      shares REAL NOT NULL,
      purchasePrice REAL NOT NULL
    )
  `);

  // GET all portfolio items
  if (request.method === 'GET' && new URL(request.url).pathname === '/portfolio') {
    const result = await sqlite.execute(`SELECT * FROM ${KEY}_portfolio_${SCHEMA_VERSION}`);
    return new Response(JSON.stringify(result.rows), {
      headers: { 'Content-Type': 'application/json' }
    });
  }

  // POST new portfolio item
  if (request.method === 'POST' && new URL(request.url).pathname === '/portfolio') {
    const asset = await request.json();
    const result = await sqlite.execute(`
      INSERT INTO ${KEY}_portfolio_${SCHEMA_VERSION} (symbol, shares, purchasePrice) 
      VALUES (?, ?, ?)
    `, [asset.symbol, asset.shares, asset.purchasePrice]);
    
    return new Response(JSON.stringify({ success: true, id: result.lastInsertRowid }), {
      headers: { 'Content-Type': 'application/json' }
    });
  }

  // PUT update portfolio item
  if (request.method === 'PUT' && new URL(request.url).pathname.startsWith('/portfolio/')) {
    const id = new URL(request.url).pathname.split('/').pop();
    const asset = await request.json();
    await sqlite.execute(`
      UPDATE ${KEY}_portfolio_${SCHEMA_VERSION} 
      SET symbol = ?, shares = ?, purchasePrice = ? 
      WHERE id = ?
    `, [asset.symbol, asset.shares, asset.purchasePrice, id]);
    
    return new Response(JSON.stringify({ success: true }), {
      headers: { 'Content-Type': 'application/json' }
    });
  }

  // DELETE portfolio item
  if (request.method === 'DELETE' && new URL(request.url).pathname.startsWith('/portfolio/')) {
    const id = new URL(request.url).pathname.split('/').pop();
    await sqlite.execute(`
      DELETE FROM ${KEY}_portfolio_${SCHEMA_VERSION} 
      WHERE id = ?
    `, [id]);
    
    return new Response(JSON.stringify({ success: true }), {
      headers: { 'Content-Type': 'application/json' }
    });
  }

  return new Response(`
    <html>
      <head>
        <title>Portfolio Tracker</title>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <style>
          body { font-family: Arial, sans-serif; line-height: 1.6; margin: 0; padding: 0; }
          * { box-sizing: border-box; }
        </style>
      </head>
      <body>
        <div id="root"></div>
        <script src="https://esm.town/v/std/catch"></script>
        <script type="module" src="${import.meta.url}"></script>
      </body>
    </html>
  `, {
    headers: { 'Content-Type': 'text/html' }
  });
}
