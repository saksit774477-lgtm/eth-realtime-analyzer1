# eth-realtime-analyzer1
# อยู่ที่โฟลเดอร์รากของรีโป (ที่เห็น README.md)
pwd

# สร้างโครงสร้างโฟลเดอร์
mkdir -p i18n .streamlit

# ---------- requirements.txt ----------
cat > requirements.txt << 'EOF'
websockets==12.0
pandas==2.2.2
numpy==1.26.4
streamlit==1.37.0
pyyaml==6.0.1
EOF

# ---------- config.yaml ----------
cat > config.yaml << 'EOF'
symbol: "ethusdt"
interval: "1m"
bars_for_calc: 300
bb_period: 21
bb_std: 2.0
rsi_period: 14
language: "th"
EOF

# ---------- indicators.py ----------
cat > indicators.py << 'EOF'
import numpy as np
import pandas as pd

def rsi(series: pd.Series, period: int = 14) -> pd.Series:
    delta = series.diff()
    gain = np.where(delta > 0, delta, 0.0)
    loss = np.where(delta < 0, -delta, 0.0)
    gain_ema = pd.Series(gain, index=series.index).ewm(alpha=1/period, adjust=False).mean()
    loss_ema = pd.Series(loss, index=series.index).ewm(alpha=1/period, adjust=False).mean()
    rs = gain_ema / loss_ema.replace(0, np.nan)
    rsi = 100 - (100 / (1 + rs))
    return rsi.fillna(50)

def bollinger(series: pd.Series, period: int = 21, std: float = 2.0):
    mid = series.rolling(period).mean()
    dev = series.rolling(period).std(ddof=0)
    up = mid + std * dev
    dn = mid - std * dev
    return up, mid, dn
EOF

# ---------- signal_rules.py ----------
cat > signal_rules.py << 'EOF'
import pandas as pd

def generate_signals(df: pd.DataFrame) -> dict:
    if len(df) < 5:
        return {"signal": None, "reason": "insufficient_bars"}

    last = df.iloc[-1]
    prev = df.iloc[-2]

    if last['close'] <= last['bb_dn'] and last['rsi'] < 30:
        return {"signal": "OVERSOLD_REBOUND_CANDIDATE", "reason": "close<=bb_dn & rsi<30"}
    if last['close'] >= last['bb_up'] and last['rsi'] > 70:
        return {"signal": "OVERBOUGHT_PULLBACK_RISK", "reason": "close>=bb_up & rsi>70"}
    if prev['close'] < prev['bb_mid'] and last['close'] > last['bb_mid'] and last['rsi'] > 50:
        return {"signal": "BULL_BB_MID_CROSS", "reason": "cross_above_mid"}
    if prev['close'] > prev['bb_mid'] and last['close'] < last['bb_mid'] and last['rsi'] < 50:
        return {"signal": "BEAR_BB_MID_CROSS", "reason": "cross_below_mid"}

    return {"signal": None, "reason": "neutral"}
EOF

# ---------- main.py (console mode) ----------
cat > main.py << 'EOF'
import asyncio, json, time, websockets, pandas as pd, yaml
from indicators import rsi, bollinger
from signal_rules import generate_signals

BINANCE_WS = "wss://stream.binance.com:9443/ws"

def stream_name(symbol, interval):
    return f"{symbol.lower()}@kline_{interval}"

def apply_indicators(df, cfg):
    df['rsi'] = rsi(df['close'], cfg['rsi_period'])
    up, mid, dn = bollinger(df['close'], cfg['bb_period'], cfg['bb_std'])
    df['bb_up'], df['bb_mid'], df['bb_dn'] = up, mid, dn
    return df

async def run(cfg):
    url = f"{BINANCE_WS}/{stream_name(cfg['symbol'], cfg['interval'])}"
    print("Connecting:", url)
    async with websockets.connect(url, ping_interval=20) as ws:
        cols = ['open_time','open','high','low','close','volume','close_time']
        df = pd.DataFrame(columns=cols)
        while True:
            k = json.loads(await ws.recv())['k']
            bar = {
                'open_time': int(k['t']),
                'open': float(k['o']),
                'high': float(k['h']),
                'low': float(k['l']),
                'close': float(k['c']),
                'volume': float(k['v']),
                'close_time': int(k['T'])
            }
            if len(df) and df.iloc[-1]['open_time'] == bar['open_time']:
                df.iloc[-1] = bar
            else:
                df.loc[len(df)] = bar
                if len(df) > cfg['bars_for_calc']:
                    df = df.iloc[-cfg['bars_for_calc']:].reset_index(drop=True)

            calc = apply_indicators(df.copy(), cfg)
            sig = generate_signals(calc)
            last = calc.iloc[-1]
            ts = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(last['close_time']/1000))
            line = f"[{ts}] close={last['close']:.2f} RSI={last['rsi']:.2f} BB={last['bb_dn']:.2f}/{last['bb_mid']:.2f}/{last['bb_up']:.2f}"
            if sig['signal']:
                print(line, "| SIGNAL:", sig['signal'], "|", sig['reason'])
            elif len(df) % 5 == 0:
                print(line)

if __name__ == '__main__':
    with open('config.yaml','r',encoding='utf-8') as f:
        cfg = yaml.safe_load(f)
    asyncio.run(run(cfg))
EOF

# ---------- streamlit_app.py (modern UI) ----------
cat > streamlit_app.py << 'EOF'
import streamlit as st, pandas as pd, asyncio, websockets, json, yaml, time, os, json as pyjson
from indicators import rsi, bollinger
from signal_rules import generate_signals

st.set_page_config(page_title="ETH Realtime Analyzer", page_icon="📈", layout="wide")
st.markdown("""
<style>
.block-container{padding-top:1.2rem;padding-bottom:1rem;}
.kpi{border-radius:14px;padding:16px 18px;background:linear-gradient(180deg,rgba(255,255,255,.05),rgba(255,255,255,.02));
     border:1px solid rgba(255,255,255,.08);box-shadow:0 6px 20px rgba(0,0,0,.25);}
.kpi .label{opacity:.75;font-size:.85rem;margin-bottom:.35rem;}
.kpi .value{font-weight:800;font-size:1.55rem;letter-spacing:.3px;}
.badge{display:inline-block;border-radius:999px;padding:4px 10px;border:1px solid rgba(255,255,255,.15);
       font-size:.78rem;margin-left:8px;}
.badge.green{background:rgba(16,185,129,.18);color:#34D399;border-color:rgba(16,185,129,.35)}
.badge.red{background:rgba(239,68,68,.18);color:#F87171;border-color:rgba(239,68,68,.35)}
.badge.yellow{background:rgba(234,179,8,.18);color:#FACC15;border-color:rgba(234,179,8,.35)}
.stTabs [data-baseweb="tab-list"]{gap:10px}
.stTabs [data-baseweb="tab"]{border-radius:10px;padding:10px 14px;background:rgba(255,255,255,.04);}
[data-testid="stMetricDelta"] svg{display:none}
</style>
""", unsafe_allow_html=True)

with open('config.yaml','r',encoding='utf-8') as f:
    cfg = yaml.safe_load(f)

def load_strings(lang):
    with open(os.path.join('i18n', f'{lang}.json'),'r',encoding='utf-8') as f:
        return pyjson.load(f)

lang = cfg.get('language','th')
strings = load_strings(lang)

with st.sidebar:
    st.header(strings['settings'])
    symbol = st.text_input(strings['symbol'], value=cfg['symbol']).lower()
    interval = st.selectbox(strings['interval'], ['1m','3m','5m','15m','1h','4h','1d'],
                            index=['1m','3m','5m','15m','1h','4h','1d'].index(cfg['interval']))
    lang_choice = st.selectbox('Language / ภาษา', ['th','en'], index=['th','en'].index(lang))
    c1, c2 = st.columns(2)
    with c1: bb_period = st.number_input("BB period", 10, 200, cfg['bb_period'])
    with c2: bb_std    = st.number_input("BB std", 1.0, 4.0, float(cfg['bb_std']), step=0.1)
    c3, c4 = st.columns(2)
    with c3: rsi_period = st.number_input("RSI period", 5, 50, cfg['rsi_period'])
    with c4: bars_keep  = st.number_input(strings['bars'], 200, 1000, cfg['bars_for_calc'], step=50)
    if st.button(strings['save']):
        cfg.update(dict(symbol=symbol, interval=interval, language=lang_choice,
                        bb_period=int(bb_period), bb_std=float(bb_std),
                        rsi_period=int(rsi_period), bars_for_calc=int(bars_keep)))
        with open('config.yaml','w',encoding='utf-8') as f:
            yaml.safe_dump(cfg, f, allow_unicode=True)
        st.success('✅ Saved. Refresh to apply.', icon="✅")

colA, colB = st.columns([0.65, 0.35])
with colA:
    st.markdown(f"### 📈 {strings['title']}  \\n*Binance WebSocket • {symbol.upper()} • {interval}*")
status_box = colB.empty()

BINANCE_WS = "wss://stream.binance.com:9443/ws"
stream = f"{symbol}@kline_{interval}"
df = pd.DataFrame(columns=['open_time','open','high','low','close','volume','close_time'])

priceKPI, rsiKPI, bandKPI = st.columns([1,1,2])
charts_tab, signals_tab, table_tab = st.tabs(["📊 Charts", "🔔 Signals", "📄 Data"])

with charts_tab:
    st.markdown("#### Price / Bands"); price_chart = st.empty()
    st.markdown("#### RSI"); rsi_chart = st.empty()
with signals_tab:
    signal_area = st.empty(); notes = st.empty()
with table_tab:
    table_area = st.empty()

def render_kpis(calc):
    last = calc.iloc[-1]
    with priceKPI:
        delta = 0.0 if len(calc)<2 else (last['close']-calc.iloc[-2]['close'])/calc.iloc[-2]['close']*100
        badge = f"<span class='badge {'green' if delta>=0 else 'red'}'>{delta:+.2f}%</span>"
        st.markdown(f"<div class='kpi'><div class='label'>{strings['current_price']}</div>"
                    f"<div class='value'>{last['close']:.2f} USDT {badge}</div></div>", unsafe_allow_html=True)
    with rsiKPI:
        r = last['rsi']; color = 'yellow' if 30<r<70 else ('green' if r>=70 else 'red')
        st.markdown(f"<div class='kpi'><div class='label'>{strings['rsi']}</div>"
                    f"<div class='value'>{r:.2f} <span class='badge {color}'>"
                    f"{'Neutral' if 30<r<70 else ('Overbought' if r>=70 else 'Oversold')}</span></div></div>", unsafe_allow_html=True)
    with bandKPI:
        st.markdown(f"<div class='kpi'><div class='label'>{strings['bbands']}</div>"
                    f"<div class='value'>{strings['bb_dn']}: {last['bb_dn']:.2f} • "
                    f"{strings['bb_mid']}: {last['bb_mid']:.2f} • "
                    f"{strings['bb_up']}: {last['bb_up']:.2f}</div></div>", unsafe_allow_html=True)

async def feed():
    status_box.info(strings['connecting'])
    async with websockets.connect(f"{BINANCE_WS}/{stream}", ping_interval=20) as ws:
        status_box.success(strings['connected'])
        global df
        while True:
            k = json.loads(await ws.recv())['k']
            bar = {'open_time':int(k['t']),'open':float(k['o']),'high':float(k['h']),
                   'low':float(k['l']),'close':float(k['c']),'volume':float(k['v']),
                   'close_time':int(k['T'])}
            if len(df) and df.iloc[-1]['open_time']==bar['open_time']:
                df.iloc[-1]=bar
            else:
                df.loc[len(df)]=bar
                if len(df)>cfg['bars_for_calc']:
                    df = df.iloc[-cfg['bars_for_calc']:]
            calc = df.copy()
            calc['rsi'] = rsi(calc['close'], cfg['rsi_period'])
            up,mid,dn = bollinger(calc['close'], cfg['bb_period'], cfg['bb_std'])
            calc['bb_up'],calc['bb_mid'],calc['bb_dn'] = up,mid,dn

            render_kpis(calc)
            price_chart.line_chart(calc[['close','bb_up','bb_mid','bb_dn']].rename(columns={'close':'Close','bb_up':'BB Up','bb_mid':'BB Mid','bb_dn':'BB Down'}))
            rsi_chart.line_chart(calc[['rsi']].rename(columns={'rsi':'RSI'}))

            sig = generate_signals(calc)
            text = strings['no_signal']; tone = "info"
            if sig['signal']=='OVERSOLD_REBOUND_CANDIDATE': text = strings['oversold'];   tone="warning"
            elif sig['signal']=='OVERBOUGHT_PULLBACK_RISK': text = strings['overbought']; tone="warning"
            elif sig['signal']=='BULL_BB_MID_CROSS':       text = strings['bull_cross_mid']; tone="success"
            elif sig['signal']=='BEAR_BB_MID_CROSS':       text = strings['bear_cross_mid']; tone="error"
            with signals_tab:
                getattr(st, 'info' if tone=='info' else ('success' if tone=='success' else ('warning' if tone=='warning' else 'error')))(f"**{strings['signal']}**: {text}")
                notes.caption(f"{strings['last_update']}: {time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(calc.iloc[-1]['close_time']/1000))} • {strings['bars']}: {len(calc)}")

            with table_tab:
                table_area.dataframe(calc.tail(120), use_container_width=True)
            await asyncio.sleep(0.05)

asyncio.run(feed())
EOF

# ---------- i18n files ----------
cat > i18n/th.json << 'EOF'
{
  "title": "ตัววิเคราะห์ ETH/USDT เรียลไทม์",
  "current_price": "ราคาปิดล่าสุด",
  "rsi": "RSI",
  "bbands": "Bollinger Bands",
  "signal": "สัญญาณ",
  "no_signal": "ยังไม่มีสัญญาณ (ภาวะปกติ)",
  "oversold": "เข้าเขตขายมากเกินไป (Oversold) อาจรีบาวน์",
  "overbought": "เข้าเขตซื้อมากเกินไป (Overbought) ระวังย่อ",
  "bull_cross_mid": "ทะลุเส้นกลาง BB ขึ้น (โมเมนตัมบวก)",
  "bear_cross_mid": "หลุดเส้นกลาง BB ลง (โมเมนตัมลบ)",
  "settings": "ตั้งค่า",
  "symbol": "สัญลักษณ์",
  "interval": "ช่วงเวลาแท่งเทียน",
  "save": "บันทึก",
  "status": "สถานะ",
  "connected": "เชื่อมต่อเรียบร้อย กำลังสตรีมข้อมูล...",
  "connecting": "กำลังเชื่อมต่อ Binance...",
  "bars": "จำนวนแท่งที่ใช้งาน",
  "bb_mid": "เส้นกลาง BB",
  "bb_up": "เส้นบน BB",
  "bb_dn": "เส้นล่าง BB",
  "last_update": "อัปเดตล่าสุด"
}
EOF

cat > i18n/en.json << 'EOF'
{
  "title": "ETH/USDT Realtime Analyzer",
  "current_price": "Last Close",
  "rsi": "RSI",
  "bbands": "Bollinger Bands",
  "signal": "Signal",
  "no_signal": "No signal (neutral)",
  "oversold": "Oversold zone – possible rebound",
  "overbought": "Overbought – pullback risk",
  "bull_cross_mid": "Crossed above BB mid (bullish momentum)",
  "bear_cross_mid": "Crossed below BB mid (bearish momentum)",
  "settings": "Settings",
  "symbol": "Symbol",
  "interval": "Candle interval",
  "save": "Save",
  "status": "Status",
  "connected": "Connected. Streaming data...",
  "connecting": "Connecting to Binance...",
  "bars": "Bars in memory",
  "bb_mid": "BB mid",
  "bb_up": "BB up",
  "bb_dn": "BB down",
  "last_update": "Last update"
}
EOF

# ---------- .streamlit theme ----------
cat > .streamlit/config.toml << 'EOF'
[theme]
base = "dark"
primaryColor = "#22D3EE"
backgroundColor = "#0B1220"
secondaryBackgroundColor = "#0F172A"
textColor = "#E2E8F0"
font = "sans serif"
EOF

echo "✅ Files created."
