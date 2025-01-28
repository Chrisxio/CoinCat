# CoinCat
Crypto coin on category search filter app
# STEP 0: INSTALL DEPENDENCIES
!pip install requests pandas plotly --quiet

# STEP 1: IMPORT LIBRARIES
import requests
import pandas as pd
import plotly.express as px
from datetime import datetime
import time

# STEP 2: API CONFIGURATION
API_KEY = "CG-YFCFjhJ9kjctxqPDJ8U6H1gq"
HEADERS = {"x-cg-api-key": API_KEY}
BASE_URL = "https://api.coingecko.com/api/v3"

# STEP 3: DYNAMIC CATEGORY HANDLING
def get_valid_categories():
    """Fetch current valid categories from API"""
    try:
        response = requests.get(f"{BASE_URL}/coins/categories/list", headers=HEADERS)
        return [cat['category_id'] for cat in response.json()]
    except Exception as e:
        print(f"⚠️ Category fetch error: {e}")
        return ['gaming', 'decentralized-finance-defi', 'layer-1']

# STEP 4: DATA FETCHING FUNCTION
def fetch_crypto_data(category='gaming', timeframe='7d'):
    """Fetch cryptocurrency data from CoinGecko API"""
    try:
        print(f"\n🔍 Fetching {timeframe} data for {category}...")

        # Validate category
        valid_cats = get_valid_categories()
        if category not in valid_cats:
            print(f"❌ Invalid category. Valid options: {valid_cats[:3]}...")
            return pd.DataFrame()

        # API parameters
        timeframe_map = {'1d': '24h', '7d': '7d', '30d': '30d'}
        params = {
            'vs_currency': 'usd',
            'category': category,
            'order': 'market_cap_desc',
            'price_change_percentage': timeframe_map[timeframe],
            'per_page': 20,
            'sparkline': 'false'
        }

        response = requests.get(f"{BASE_URL}/coins/markets",
                              headers=HEADERS,
                              params=params)
        response.raise_for_status()

        df = pd.DataFrame(response.json())

        if not df.empty:
            print(f"✅ Found {len(df)} coins")
            return df
        else:
            print("❌ No data found")
            return pd.DataFrame()

    except Exception as e:
        print(f"🔥 Data fetch failed: {e}")
        return pd.DataFrame()

# STEP 5: SCORING CALCULATION
def calculate_scores(df, timeframe='7d'):
    """Calculate coin scores"""
    try:
        print("\n🧮 Calculating scores...")

        # Get correct price change column
        time_suffix = '24h' if timeframe == '1d' else timeframe
        price_change_col = f'price_change_percentage_{time_suffix}'

        # Create missing columns if needed
        if price_change_col not in df.columns:
            df[price_change_col] = 0.0

        # Calculate metrics
        df['price_change'] = df[price_change_col].fillna(0)
        df['volume_ratio'] = (df['total_volume'] / df['market_cap']).fillna(0)
        df['score'] = (df['price_change'] * 0.5) + (df['volume_ratio'] * 0.5)

        return df.sort_values('score', ascending=False).head(20)

    except Exception as e:
        print(f"🧮 Scoring failed: {e}")
        return pd.DataFrame()

# STEP 6: MAIN ANALYSIS FUNCTION
def analyze_crypto(category='gaming', timeframe='7d'):
    """Main analysis workflow"""
    print(f"\n{'='*40}")
    print(f"🚀 Starting analysis: {category} ({timeframe})")
    print(f"{'='*40}")

    # Fetch data
    df = fetch_crypto_data(category, timeframe)

    if not df.empty:
        # Calculate scores
        ranked_df = calculate_scores(df, timeframe)

        if not ranked_df.empty:
            # Show results
            print("\n🏆 Top Coins:")
            display(ranked_df[['name', 'symbol', 'current_price', 'score']]
                    .style.format({'current_price': '${:.4f}', 'score': '{:.2f}'}))

            # Create visualization
            print("\n📊 Generating chart...")
            fig = px.scatter(ranked_df,
                            x='volume_ratio',
                            y='price_change',
                            size='market_cap',
                            color='score',
                            hover_name='name',
                            title=f'{category} Coin Analysis')
            fig.show()

            # Save results
            timestamp = datetime.now().strftime("%Y%m%d_%H%M")
            filename = f"{category}_{timeframe}_{timestamp}.csv"
            ranked_df.to_csv(filename, index=False)
            print(f"\n💾 Saved results to {filename}")
        else:
            print("\n❌ No coins met ranking criteria")
    else:
        print("\n❌ Analysis failed - no data available")

# STEP 7: RUN PROGRAM
if __name__ == "__main__":
    # Test API connection first
    print("🔌 Testing API connection...")
    try:
        test = requests.get(f"{BASE_URL}/ping", headers=HEADERS)
        if test.json().get('gecko_says') == '(V3) To the Moon!':
            print("✅ API Connection Successful!")
        else:
            print("❌ API Connection Failed")
    except Exception as e:
        print(f"🔌 Connection test failed: {e}")

    time.sleep(1)  # Rate limit buffer

    # Run analysis (MODIFY THESE VALUES)
    analyze_crypto(category='gaming', timeframe='1d')
    # analyze_crypto(category='layer-1', timeframe='30d')
    # analyze_crypto(category='decentralized-finance-defi', timeframe='1d')
