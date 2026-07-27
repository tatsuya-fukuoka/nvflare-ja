.. _industry_use_cases:

########################################
業界ユースケース
########################################

連合学習(Federated Learning)は、プライバシー、規制、または競争上の制約によりデータを一元化できない
さまざまな業界で採用が進んでいます。NVIDIA FLAREは、こうしたデプロイメントを実現するための
プラットフォームインフラストラクチャを提供します。

ヘルスケアとライフサイエンス
============================================================

連合学習により、患者データを共有することなく、医療AIモデル開発のための
複数病院間のコラボレーションが可能になります。

**主なアプリケーション:**

- 医用画像解析(放射線科、病理、眼科)
- 創薬および分子特性予測
- 電子健康記録(EHR)の解析
- 臨床試験の最適化
- ゲノミクスと精密医療

**なぜ連合学習なのか?** HIPAA、患者の同意、機関のデータガバナンスポリシーにより、
データの一元的な集約は妨げられています。連合学習を使えば、病院は患者データをローカルに
保持したまま、共同でより良いモデルをトレーニングできます。

**Cancer AI Alliance (CAIA):**
主要ながんセンターのコンソーシアムであるCancer AI Allianceは、NVIDIA FLAREを
Rhino Federated Computing Platformとともに使用し、機密性の高い患者データを各センターの
ファイアウォールの内側に保持したまま、複数の機関にまたがってAIモデルをトレーニングしています。
交換されるのはモデル重みのみであり、がんセンターはデータの選択、アクセスポリシー、
ローカル実行に対する完全な制御を保持します。プラットフォームは、より高度な保護を必要とするプロジェクト向けに、差分プライバシーとモデル暗号化をサポートしています。

- `How CAIA operationalizes secure, multi-site research with federated learning (Dec 2025) <https://www.canceralliance.ai/blog/caia-multi-site-research-federated-learning>`_
- `Federating cancer research at scale (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd08?playlistId=playList-eacb3be4-9f4b-48d0-98fc-f7a40f93d759>`_

**Eli Lilly TuneLab -- 連合創薬:**
2025年9月、Eli Lillyは、10億ドルを超えるLillyの研究データでトレーニングされた創薬モデルへの
アクセスをバイオテクノロジー企業に提供するAI/MLプラットフォーム、TuneLabを立ち上げました。
このプラットフォームは連合学習を使用しており、バイオテクノロジーのパートナーは、自社の独自の
分子データを公開することなく、そのデータ上でLillyのモデルをファインチューニングできます。その見返りとして、
パートナーはトレーニングデータを提供し、エコシステム全体の共有モデルを継続的に改善します。

- `Lilly launches TuneLab platform (Sep 2025) <https://investor.lilly.com/news-releases/news-release-details/lilly-launches-tunelab-platform-give-biotechnology-companies>`_

**Federated AI for Therapeutic Engineering (FAITE) -- AbbVie、Amgen、AstraZeneca、J&J、UCB:**
2025年に発足したFAITEは、バイオ医薬品の特性予測モデルを連合学習と能動学習によってトレーニングする、
業界横断的なバイオ医薬品コンソーシアムです。メンバー企業は、ローカルの独自の分子データを
共有することなくトレーニングに貢献し、競争上および規制上のデータ境界を維持しながら
協調的なモデル改善を実現しています。

- `FAITE: Federated AI for biologics property prediction (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd13?playlistId=playList-eacb3be4-9f4b-48d0-98fc-f7a40f93d759>`_
- `Training federated AI models to predict protein properties (NVIDIA Blog) <https://developer.nvidia.com/blog/training-federated-ai-models-to-predict-protein-properties/>`_

**その他の参考資料:**

- `Federated Learning for Brain Tumor Segmentation (Nature Communications) <https://doi.org/10.1038/s41467-022-33407-5>`_
-  さらなるユースケースはFLARE DAYの録画をご覧ください

金融サービス
============================

金融機関は、機密性の高い取引データを公開することなく、不正検知、信用リスクモデリング、
マネーロンダリング対策のために連合学習を活用しています。

**主なアプリケーション:**

- 決済ネットワークをまたぐ不正検知
- より広範なデータ表現による信用スコアリング
- マネーロンダリング対策(AML)モデルのトレーニング
- 市場リスク分析

**なぜ連合学習なのか?** 銀行規制(SOX、FINRA、GDPR、PSD2)と競争上の機密性により、
機関間での取引データの共有は妨げられています。

**Swift Collaborative Fraud Defence:**
2025年9月、SwiftはANZ、BNY、Intesa Sanpaoloを含む世界の13の銀行と提携し、
国境を越えた不正検知のための連合学習をテストしました。プライバシー強化技術を用いて、
参加機関は顧客情報を共有することなく、自らのデータ上でローカルにAIモデルをトレーニングしました。
1,000万件の人工取引を対象とした試験では、協調的な連合モデルは、単一機関のデータのみで
トレーニングされたモデルと比較して、既知の不正取引の検知において **2倍の効果** を示しました。

**参考資料:**

- `Swift-led experiments reveal blueprint for collaborative fraud defence using AI (Sep 2025) <https://www.swift.com/news-events/news/swift-led-experiments-reveal-blueprint-collaborative-fraud-defence-using-ai>`_
- `Federated fraud detection at Swift (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd09?playlistId=playList-eacb3be4-9f4b-48d0-98fc-f7a40f93d759>`_

**JP Morgan、BNY、RBC -- 連合金融AI:**
GTC 2025において、JP Morgan、BNY、Royal Bank of Canada (RBC)は、金融AIモデルへの連合学習の
適用経験を発表しました。機密性の高い顧客取引データを共有することなく、リスクおよび不正の
ユースケースに対して機関横断的なモデルトレーニングを行った内容が紹介されています。

- `Federated Learning in Financial Services: JP Morgan, BNY, RBC (GTC 2025) <https://www.nvidia.com/en-us/on-demand/session/gtcdc25-dc51038/?playlistId=playList-fd045586-2409-4d1a-8333-e1d3501d52de>`_

政府と国家安全保障
====================================

政府機関や国立研究所は、機密区分、データ主権、またはセキュリティ上の制約により一元化できない、
地理的に分散した機密データセット上でAIモデルを協調的にトレーニングするために
連合学習を活用しています。

**Trilab連合AI (Sandia、Los Alamos、Lawrence Livermore):**
2025年、NNSAの3つの国家安全保障研究所は、生データを交換することなく、地理的に分散した
3つの機密システムをまたいで共有の大規模言語モデルをトレーニングする連合学習プロトタイプ
(コードネーム *Chandler*)を実証しました。
NVIDIA FLAREを使ってトレーニングをオーケストレーションし、各研究所固有のデータセットを
ローカルに保持したまま、エポック間でモデル重み(パラメーター)のみを交換します。このプロトタイプは、
世界最速のスーパーコンピューターであるLawrence LivermoreのEl Capitanを含む、
NVIDIAとAMDの両方のGPUハードウェア上で実行されました。

*"Federated training is a critical tool to delivering a robust capability in a cost
effective, performant and secure way."* -- Si Hammond, NNSA Office of Advanced Simulation
and Computing

**参考資料:**

- `Three national security laboratories, one AI model (Sandia Lab News, Dec 2025) <https://www.sandia.gov/labnews/2025/12/18/three-national-security-laboratories-one-ai-model/>`_
- `Trilab federated LLM training across classified systems (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd18?playlistId=playList-eacb3be4-9f4b-48d0-98fc-f7a40f93d759>`_

**Oak Ridge National Laboratory (ORNL) -- OLCF科学研究:**
Oak Ridge Leadership Computing Facility (OLCF)は、分散した科学データセットをまたぐ連合学習に
NVIDIA FLAREを使用しており、機密性の高い実験データを一元化することなく、大規模科学計算研究における
マルチサイトコラボレーションを実現しています。

- `ORNL OLCF: Federated learning for scientific research (FLARE Day 2024) <https://developer.download.nvidia.com/assets/Clara/flare/NVFLARE_DAY_2024_Part_09_ORNL.mp4>`_

**台湾国際連合学習センター(衛生福利部):**
2026年1月、台湾の衛生福利部は、データのプライバシーと主権を守りながらスマート医療AIモデルを
トレーニングするための国際ハイコンピューティング・連合学習センターの設立を発表しました。
同センターは16の主要病院との概念実証を完了しており、100の地域病院、最終的には台湾のすべての
病院への拡大を計画しています。医療データはローカルの病院サーバーに保持されたまま、中央のAI
モデルが連合学習を通じて分散データから学習します。同センターはまた、タイのMahidol University
との国際協力を進めており、ASEAN市場全体でAIベースの医療製品検証の標準を共同開発しています。

**参考資料:**

- `Taiwan launches new era of medical AI with global collaboration (Jan 2026) <https://www.rti.org.tw/en/news?uid=3&pid=188971>`_
- `Taiwan International Federated Learning Center (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd23?playlistId=playList-4dd15c3d-2422-425c-b9d9-d21694397574>`_


交通・運輸
====================

**自動運転車のための連合学習:**
自動車メーカーや研究チームは、分散した車両フリートやテスト施設をまたいで認識モデルや安全モデルを
トレーニングするために連合学習を活用しており、独自の走行データを一元化することなく
モデル品質を向上させています。

- `Federated Learning for Autonomous Vehicles (FLARE Day 2024) <https://developer.download.nvidia.com/assets/Clara/flare/NVFLARE_DAY_2024_Part_02_AV.mp4>`_


エッジAIと科学計算
====================================

**NVIDIA Holoscanによるエッジでの連合分析:**
NVIDIA Holoscanは、医療機器や産業用AIアプリケーション向けにエッジでの連合分析を可能にし、
生のセンサーストリームを中央サーバーに送信することなく、推論データを連合モデルの改善に
活用できるようにします。

- `Holoscan Federated Analytics at the Edge (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd26?playlistId=playList-4dd15c3d-2422-425c-b9d9-d21694397574>`_

**NVIDIA Data Federation Mesh -- 科学計算における連合データ処理:**
NVIDIA Data Federation Meshは、大規模科学計算向けの連合データ処理パイプラインを実証しており、
生の科学データセットを移動させることなく、施設をまたいだ分散解析を可能にします。

- `NVIDIA Data Federation Mesh (FLARE Day 2025) <https://www.nvidia.com/en-us/on-demand/session/nvidiaflareday25-nvfd15?playlistId=playList-eacb3be4-9f4b-48d0-98fc-f7a40f93d759>`_


FLARE Day -- 実世界のデプロイメント
========================================================================

FLARE Dayは、ヘルスケア、金融、自動運転などにおける実世界の連合学習デプロイメントを紹介する
年次イベントです。これらの講演では、実務者が本番環境での経験と得られた教訓を
共有しています。

- **FLARE Day 2026** -- *2026年9月開催予定*
- `FLARE Day 2025 <https://developer.nvidia.com/flare-day-2025>`_ -- ヘルスケア、金融、自動運転などにおける実世界のFLアプリケーション
- `FLARE Day 2024 <https://nvidia.github.io/NVFlare/flareDay>`_ -- NVIDIA、医療機関、業界パートナーによる実世界のFLデプロイメントを紹介する講演とデモ

自分の業界で始めるには
============================================

どの業界であっても、連合学習への道のりは同様のパターンをたどります:

1. **ユースケースを特定する** -- 連合データで改善したいMLモデルは何ですか?
2. **シミュレーションから始める** -- :ref:`FL Simulator <fl_simulator>` を使って合成データでプロトタイピングします
3. **POCで価値を実証する** -- 2〜3の参加サイトで :ref:`POCデプロイメント <poc_command>` を実行します
4. **本番環境へスケールする** -- プロビジョニングとインフラストラクチャについては :doc:`デプロイメントガイド <user_guide/admin_guide/deployment/overview>` に従います

業界固有のデプロイメントに関する質問については、:doc:`publications_and_talks` ページで
各分野に関連する講演や論文を参照してください。
