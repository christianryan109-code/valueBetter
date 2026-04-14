# Odds edge
is a high-frequency odds scraping engine designed to identify Positive Expected Value (+EV) opportunities by identifying "slow" lines on soft sportsbooks compared to sharp market makers.

## How it Works
The system operates on the principle of Market Convergence.

Scrape: Connects via WebSockets/APIs to "Sharp" books (e.g., Pinnacle, Betfair) to establish the "Fair Price."

Calculate: Removes the sportsbook margin (the vig) to find the true mathematical probability of an outcome.

Compare: Scans crypto-native books (e.g., Stake, Rollbit, Polymarket) to find odds that are significantly higher than the Fair Price.

Execute: Identifies the edge percentage and calculates optimal stake using the Kelly Criterion.

Note: This project is for educational purposes. Automated betting may violate the Terms of Service of certain platforms.
