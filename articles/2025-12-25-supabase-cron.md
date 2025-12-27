---
title: "Supabase の DB 拡張だけで作る定期集計 Slack 通知"
emoji: "🐈‍⬛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Supabase", "pg_cron", "pg_net"]
publication_name: "japagate"
published: true
---

:::message
この記事は、[ジャパゲートシステムズ Advent Calendar 2025](https://qiita.com/advent-calendar/2025/japagate-systems) 25 日目の記事です。
:::

DB にためられたコンバージョンデータを daily で集計して Slack に通知する仕組みを構築したのですが、その際に supabase の機能を使うとめっちゃ簡単にできたので共有してみます。

## まとめ

- 定期集計してSlackに通知する仕組みを **SupabaseのDB拡張だけ** で作った
  - [pg_cron](https://supabase.com/docs/guides/database/extensions/pg_cron) で定期実行
  - [pg_net](https://supabase.com/docs/guides/database/extensions/pg_net) で Slack に直接 Webhook 通知
- Edge Function 等は一切使っていない
- 小さな定期レポート用途なら DB 主導の構成が一番シンプルかも
- 注意点
  - pg_cron は 2025/12/28 現在 beta 版としての提供されており、本番運用には注意が必要
  - デバッグがしにくくなったり、ログの吐き方などが不便な点もある

## やったこと

定期レポーティング／サマリー通知の仕組みを構築したい際、よくある構成だと『集計 + 通知用の batch function をつくり、それを外部 cron を使って定期実行する』とかがあると思います。
今回は supabase の機能だけで集計 + 通知を実現する仕組みを構築しました。

### pg_net で slack へ通知する関数を作る

複雑なロジックはないので詳細は割愛しますが、以下の関数を定義しました。

ここでは『セグメントごとのトラッキング数を集計して Slack に通知する』というシンプルな仕組みを構築しました。

``` sql
CREATE OR replace FUNCTION public.post_slack_report()
RETURNS void
LANGUAGE plpgsql
SECURITY definer
-- webhook_url を secret という schema に格納しているため。
SET search_path = public, secret
AS $$
DECLARE
  message_text TEXT;
  slack_payload JSONB;
  period_start TEXT;
  period_end TEXT;
BEGIN
  -- =====================
  -- 欲しい情報を SQL で集計する。
  -- =====================
  SELECT string_agg(
    room_name || ' => ' || cnt::text,
    E'\n'
    ORDER BY cnt DESC
  )
  FROM (
    -- 1日の集計結果をダミーデータで作成
    VALUES
      ('ルームA', 42),
      ('ルームB', 28),
      ('ルームC', 15),
      ('(未設定)', 3)
  ) AS dummy_within_one_day(room_name, cnt)
  INTO message_text;

  -- =====================
  -- Slack blocks 形式で payload を構築する。
  -- =====================
  slack_payload := jsonb_build_object(
    'blocks', jsonb_build_array(
      -- ヘッダー: 集計期間
      jsonb_build_object(
        'type', 'header',
        'text', jsonb_build_object(
          'type', 'plain_text',
          'text', 'トラッキングレポート (' || period_start || ' ~ ' || period_end || ')'
        )
      ),
      -- 本文: 部屋ごとのイベント数
      jsonb_build_object(
        'type', 'section',
        'text', jsonb_build_object(
          'type', 'mrkdwn',
          'text', COALESCE(message_text, '(データなし)')
        )
      ),
      -- フッター: 詳細リンク
      jsonb_build_object(
        'type', 'context',
        'elements', jsonb_build_array(
          jsonb_build_object(
            'type', 'mrkdwn',
            'text', '<link_to_supabase_sql_editor|詳細を確認する> | <link_to_cron_document|cron詳細>'
          )
        )
      )
    )
  );


  -- =====================
  -- pg_net を使って slack へ http request で送る。
  -- =====================
  perform net.http_post(
    url     := secret.get_slack_webhook_url(),
    headers := '{"Content-Type":"application/json"}',
    body    := slack_payload
  );
END;
$$;
```

ここでは [net.http_post](https://supabase.com/docs/guides/database/extensions/pg_net#httppost) を使って直接 Slack の Webhook 用エンドポイントを叩いています。
詳しい使い方は [公式ドキュメント](https://supabase.com/docs/guides/database/extensions/pg_net#enable-the-extension) を参照してください。
（Extension を有効にするだけで使えるようになります。）

### pg_cron を使って定期実行する

[pg_cron](https://supabase.com/docs/guides/cron) も supabase の Extension の 1 つで、2025/12/28 現在以下の 4 つのパターンに対応してます。

- SQL snipet の実行
- Database 関数の実行
- HTTP エンドポイントの呼び出し
- Supabase Edge Function の呼び出し

![](/images/2025-12-25-supabase-cron/pg_cron.png)

今回は以下のようなコードを実行させることで、Database 関数を定期実行する仕組みを構築しました。

``` sql
-- Cron 登録。
SELECT cron.schedule(
  'daily-report-1500-jst',
  '0 6 * * *',
  $$ SELECT public.post_slack_report(); $$
);
```

## おわりに

Supabase の DB 拡張には、ベースとする Postgres の持つ拡張機能や固有の拡張も含めて[かなりの数用意されています](https://supabase.com/docs/guides/database/extensions)。
（wrap した普通の機能として提供されてるものもある）

以下に自分が気になってるものを挙げておきますので、みなさんのおすすめや気になりも是非教えてください！

- Realtime
  - DBの変更をリアルタイムに購読できる
- pgjwt
- pgvector
- pg_graphql
