# NVIDIA FLARE 日本語ドキュメント(非公式)

[NVIDIA FLARE](https://github.com/NVIDIA/NVFlare) の[公式ドキュメント](https://nvflare.readthedocs.io/en/main/index.html)を日本語に翻訳した**非公式**ドキュメントサイトです。

公開サイト: **https://tatsuya-fukuoka.github.io/nvflare-ja/**

## 本プロジェクトについて

- 翻訳のベース: [NVIDIA/NVFlare](https://github.com/NVIDIA/NVFlare) `main` ブランチの `docs/` ディレクトリ
  - ベースコミット: `2c63764cce36a98104701dc297f467a8aa3f5da6` (2026-07時点)
- **全 214 ページの日本語翻訳が完了しています。** ページ単位の一覧は [TRANSLATION_STATUS.md](TRANSLATION_STATUS.md) を参照してください。
- コードブロック、コマンド、ファイルパス、設定キー名、クラス名などの識別子は原文のまま保持しています。
- APIリファレンス(apidocs)は自動生成のため翻訳対象外とし、[公式APIリファレンス](https://nvflare.readthedocs.io/en/main/apidocs/modules.html)へのリンクとしています。
- 翻訳内容と原文に差異がある場合は、常に[公式ドキュメント(英語)](https://nvflare.readthedocs.io/en/main/index.html)が優先されます。

## ビルド方法

```bash
pip install -r requirements.txt
SKIP_API_DOCS=1 sphinx-build -b html docs docs/_build/html
```

ビルド結果は `docs/_build/html/index.html` から閲覧できます。

## デプロイ

`main` ブランチへのpushをトリガーに、GitHub Actions([.github/workflows/deploy.yml](.github/workflows/deploy.yml))がSphinxビルドとGitHub Pagesへのデプロイを実行します。

初回のみ、リポジトリの Settings → Pages → Build and deployment → Source を **GitHub Actions** に設定する必要があります。

## ライセンス

原文ドキュメントおよび同梱のソースコードは NVIDIA CORPORATION による [Apache License 2.0](LICENSE) の下で提供されています。本翻訳も同ライセンスに従います。

- 原文: Copyright (c) NVIDIA CORPORATION
- 本リポジトリは NVIDIA とは無関係の非公式翻訳プロジェクトです
