# Web Speed Hackathon 2026 - クライアントサイド最適化計画

## Context
Web Speed Hackathon 2026 の SNS アプリ「CaX」のクライアントサイドパフォーマンスを最適化する。意図的に重くされたアプリケーションのボトルネックを特定し、Lighthouse スコア（1150点満点）を最大化する。

**スコア配分:** ページ表示 900pts (9ページ × FCP×10 + SI×10 + LCP×25 + TBT×30 + CLS×25) + ページ操作 250pts (5シナリオ × TBT×25 + INP×25)

**重要メトリクス重み:** TBT(30%) > LCP(25%) = CLS(25%) > FCP(10%) = SI(10%)

**制約:** E2Eテスト・VRTを壊さない / レギュレーション遵守 / Crok SSEプロトコル変更禁止

---

## 特定されたボトルネック一覧

### A. ビルド設定の問題
| # | 問題 | ファイル | 影響メトリクス |
|---|------|---------|--------------|
| A1 | webpack mode:"none", 全最適化無効 | `client/webpack.config.js:39,130-137` | FCP,SI,LCP,TBT |
| A2 | devtool: "inline-source-map" (バンドルサイズ倍増) | `client/webpack.config.js:28` | FCP,SI,LCP |
| A3 | Babel targets: "ie 11" (大量polyfill) | `client/babel.config.js:6` | FCP,SI,LCP,TBT |
| A4 | Babel modules: "commonjs" (tree shaking不可) | `client/babel.config.js:8` | FCP,SI,LCP |
| A5 | React preset development: true | `client/babel.config.js:15` | TBT |
| A6 | エントリに core-js, regenerator-runtime を丸ごとinclude | `client/webpack.config.js:30-32` | FCP,SI |
| A7 | NODE_ENV: "development" | `client/webpack.config.js:78` | TBT |

### B. レンダリングブロック
| # | 問題 | ファイル | 影響メトリクス |
|---|------|---------|--------------|
| B1 | `<script>` が defer/async なしで `<head>` に配置 | `client/src/index.html:7` | FCP,SI,LCP |
| B2 | Tailwind CSS を CDN から JS としてランタイム実行 | `client/src/index.html:9` | FCP,SI,LCP |
| B3 | React マウントが `window.load` イベント待ち | `client/src/index.tsx:8` | FCP,SI,LCP |
| B4 | font-display: block (フォントロードまでテキスト非表示) | `client/src/index.css:8,15` | FCP,LCP |
| B5 | OTFフォント (WOFF2より大きい) | `client/src/index.css:9,16` | FCP,LCP |

### C. メインスレッドブロック
| # | 問題 | ファイル | 影響メトリクス |
|---|------|---------|--------------|
| C1 | 全AJAX呼び出しが `async: false` (同期XHR) | `client/src/utils/fetchers.ts:6,17,28,46` | TBT,INP |
| C2 | InfiniteScroll: 2^18回(262,144回)のDOM読取り/スクロールイベント | `client/src/components/foundation/InfiniteScroll.tsx:17` | TBT,INP |
| C3 | jQuery + pako による gzip 圧縮 | `client/src/utils/fetchers.ts:42-43` | TBT |

### D. コンポーネントの問題
| # | 問題 | ファイル | 影響メトリクス |
|---|------|---------|--------------|
| D1 | CoveredImage: 画像をバイナリDL→EXIF解析→サイズ計算 | `client/src/components/foundation/CoveredImage.tsx` | TBT,LCP |
| D2 | AspectRatioBox: setTimeout(500ms) でCLS発生 | `client/src/components/foundation/AspectRatioBox.tsx:22` | CLS,LCP |
| D3 | PausableMovie: GIFをバイナリDL→JSでフレームデコード | `client/src/components/foundation/PausableMovie.tsx` | TBT,LCP |

### E. バンドルサイズ (コード分割なし)
| # | 問題 | ファイル | 影響メトリクス |
|---|------|---------|--------------|
| E1 | 全ルートを即座にimport (lazy loadingなし) | `client/src/containers/AppContainer.tsx:5-16` | FCP,SI,LCP,TBT |
| E2 | 巨大依存: @ffmpeg, @imagemagick, @mlc-ai/web-llm | 各utils/ | FCP,TBT |
| E3 | moment.js (全locale込み ~300KB) | 複数ファイル | FCP |
| E4 | lodash (フルバンドル ~70KB) | utils/, components/ | FCP |
| E5 | jQuery (fetch APIで代替可能 ~90KB) | utils/fetchers.ts | FCP |
| E6 | standardized-audio-context (ネイティブAPIで十分) | webpack ProvidePlugin | FCP |

---

## 最適化フェーズ (実施順)

### Phase 1: ビルド設定修正 (影響: 最大) ✅ 完了
1. ✅ `webpack.config.js`: mode→"production", devtool→false, optimization全有効化 (minimize, concatenateModules, usedExports, providedExports, sideEffects)
2. ✅ `babel.config.js`: targets→"last 2 Chrome versions", modules→false, useBuiltIns→"entry"
3. ✅ `webpack.config.js`: NODE_ENV→"production", React development→false
4. ⏭️ エントリのcore-js, regenerator-runtimeは保持（useBuiltIns: "entry"で最適化済み。削除するとランタイムエラーのリスクあり）

**ビルド結果:** 成功 (35.8秒) / main.js: 72.1MB / レギュレーション違反なし

### Phase 2: レンダリングブロック解消 (影響: 大)
1. `index.html`: script に defer 追加、HtmlWebpackPlugin inject設定
2. Tailwind CSS をビルド時PostCSS統合に移行、CDN script 削除
3. `index.tsx`: window.load 待ちを削除、直接マウント
4. `index.css`: font-display→swap、フォントをWOFF2に変換

### Phase 3: メインスレッドブロック解消 (影響: 大)
1. `fetchers.ts`: jQuery sync AJAX → native fetch API
2. `InfiniteScroll.tsx`: IntersectionObserver に置換
3. jQuery, pako, jquery-binarytransport 依存削除

### Phase 4: コンポーネント修正 (影響: 中)
1. `CoveredImage.tsx`: `<img>` + `object-fit:cover` + サーバーからalt取得
2. `AspectRatioBox.tsx`: CSS `aspect-ratio` プロパティに置換
3. `PausableMovie.tsx`: 遅延ロード or video要素活用

### Phase 5: コード分割 (影響: 中)
1. `AppContainer.tsx`: React.lazy() でルート分割
2. Crok関連 (katex, react-markdown, web-llm等) を動的import
3. 投稿作成関連 (ffmpeg, imagemagick) を動的import
4. 検索関連 (kuromoji, negaposi-analyzer) を動的import

### Phase 6: 依存関係の軽量化 (影響: 小〜中)
1. moment.js → dayjs or Intl API
2. lodash → ネイティブJS
3. jQuery 完全削除 (Phase 3後)
4. standardized-audio-context 削除

---

## 検証方法
1. `pnpm build` でビルド成功を確認
2. E2Eテスト: `cd /application/e2e && pnpm test` で全テスト通過
3. Lighthouse: scoring-tool でスコア計測
4. バンドルサイズ: webpack-bundle-analyzer で前後比較
