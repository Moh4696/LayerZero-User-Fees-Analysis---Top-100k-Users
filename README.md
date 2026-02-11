# LayerZero-User-Fees-Analysis---Top-100k-Users
# Query 1: User leaderboard ranked by total fees paid
# Shows all-time fees, pre-snapshot fees, and post-snapshot fees

WITH user_fees AS (
    SELECT 
        addr_from AS user_address
        , COUNT(*) AS total_transactions
        
        -- All-time fee metrics (DVN fee + Executor fee)
        , SUM(COALESCE(usd_dvn_fee, 0) + COALESCE(usd_executor_fee, 0)) AS all_time_fees_usd
        
        -- Pre-snapshot fees (before May 1, 2024 - counted for 1st airdrop)
        , SUM(CASE WHEN ts_source < TIMESTAMP '2024-05-01 00:00:00' 
              THEN COALESCE(usd_dvn_fee, 0) + COALESCE(usd_executor_fee, 0) ELSE 0 END) AS pre_snapshot_fees_usd
        , COUNT(CASE WHEN ts_source < TIMESTAMP '2024-05-01 00:00:00' THEN 1 END) AS pre_snapshot_tx_count
        
        -- Post-snapshot fees (May 1, 2024 onwards - potential 2nd airdrop)
        , SUM(CASE WHEN ts_source >= TIMESTAMP '2024-05-01 00:00:00' 
              THEN COALESCE(usd_dvn_fee, 0) + COALESCE(usd_executor_fee, 0) ELSE 0 END) AS post_snapshot_fees_usd
        , COUNT(CASE WHEN ts_source >= TIMESTAMP '2024-05-01 00:00:00' THEN 1 END) AS post_snapshot_tx_count
        
        -- Post-redistribution fees (September 20, 2024 onwards - for 2nd airdrop eligibility)
        , SUM(CASE WHEN ts_source >= TIMESTAMP '2024-09-20 00:00:00' 
              THEN COALESCE(usd_dvn_fee, 0) + COALESCE(usd_executor_fee, 0) ELSE 0 END) AS post_redistribution_fees_usd
        , COUNT(CASE WHEN ts_source >= TIMESTAMP '2024-09-20 00:00:00' THEN 1 END) AS post_redistribution_tx_count
        
        -- Time metrics
        , MIN(ts_source) AS first_transaction_date
        , MAX(ts_source) AS last_transaction_date
        
    FROM layerzero.messages
    WHERE addr_from IS NOT NULL
    GROUP BY 1
)

SELECT 
    ROW_NUMBER() OVER (ORDER BY all_time_fees_usd DESC) AS rank
    , user_address
    
    -- All-time metrics
    , total_transactions AS total_txs
    , ROUND(all_time_fees_usd, 2) AS total_fees_usd
    , ROUND(all_time_fees_usd / NULLIF(total_transactions, 0), 4) AS avg_fee_per_tx_usd
    
    -- Pre-snapshot (1st airdrop eligibility period)
    , pre_snapshot_tx_count AS pre_snap_txs
    , ROUND(pre_snapshot_fees_usd, 2) AS pre_snap_fees_usd
    , ROUND(pre_snapshot_fees_usd / NULLIF(all_time_fees_usd, 0) * 100, 1) AS pct_fees_pre_snapshot
    
    -- Post-snapshot (potential 2nd airdrop)
    , post_snapshot_tx_count AS post_snap_txs
    , ROUND(post_snapshot_fees_usd, 2) AS post_snap_fees_usd
    , ROUND(post_snapshot_fees_usd / NULLIF(all_time_fees_usd, 0) * 100, 1) AS pct_fees_post_snapshot
    
    -- Post-redistribution (Sep 20, 2024+)
    , post_redistribution_tx_count AS post_redist_txs
    , ROUND(post_redistribution_fees_usd, 2) AS post_redist_fees_usd
    , ROUND(post_redistribution_fees_usd / NULLIF(all_time_fees_usd, 0) * 100, 1) AS pct_fees_post_redist
    
    -- Fee comparison
    , ROUND(post_snapshot_fees_usd - pre_snapshot_fees_usd, 2) AS fee_change_usd
    
    -- Timestamps
    , first_transaction_date
    , last_transaction_date
    
FROM user_fees
WHERE all_time_fees_usd > 0  -- Only users who paid fees
ORDER BY all_time_fees_usd DESC
LIMIT 100000
