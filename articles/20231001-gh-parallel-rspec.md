---
title: "月$12を払うか、1700分を浮かすか - テストを実行時間ベースで分割し並列実行する話 | Offers Tech Blog"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rspec", "ruby", "GithubActions"]
published: false
publication_name: overflow_offers
---

[offersurl]: https://offers.jp
[overflow]: https://overflow.co.jp

こんにちは、プロダクト開発人材の副業転職プラットフォーム [Offers][offersurl] を運営する株式会社 [overflow][overflow] CTO の大谷旅人です。
今回は、CI(GitHub Actions)上のテストにかかる時間が長く、リリース時にテスト完了待ちで無駄な時間が発生しているというありがちな問題に対して、ワークフローだけ弄って 3 分でテスト時間半分にした結果改善されたのでそんなお話を。

# リリースまでのフローと課題

参考までにその PJT でのリリースまでのフローは以下のような流れでした。

![](https://storage.googleapis.com/zenn-user-upload/93d736d7746d-20231001.png)

1. feature ブランチでのチェック完了後、develop ブランチ(メインライン) にマージ
2. develop ブランチ(メインライン) にマージ後、以下が実行
  2.2. SmokeTest 環境へのデプロイと、E2E 実行
  2.3. Backend の UnitTest(RSpec)/Frontend の StaticTest 実行
3. すべてのチェックが通過後、ReleasePR が作成され main ブランチに反映しリリース

![](https://storage.googleapis.com/zenn-user-upload/6d53e1301764-20231001.png)

この中の `Backend の UnitTest(RSpec)` が、モデル & ロジック数の増加によって 35min 以上かかるようになってきたという状況です。


# 対応

このように、テスト時間が長い時に対応として考えることは、以下です。

- テストの並列実行、テストケースの分割
- 実行インスタンスのスケールアップとコスト算定
  - [Larger Runner](https://docs.github.com/en/actions/using-github-hosted-runners/about-larger-runners/about-larger-runners) の検討
- ステージ間の依存関係を最適化
  - ビルドキャッシュが有効につかえるか等
- テストスイートの分割
  - 変更箇所に関連するテストのみを実行するなど
- 不要なテストの削除、モック or スタブ活用しての短縮が可能か
  - 過度なテストや、無駄なテストデータ生成が行われてないか

今回は、`テストの並列実行、テストケースの分割` をとりあえず試したら、案外うまくいったという結果でした。
(テストはメンテを怠ると、すぐに負債化するので、定期的な見直しが重要ですね..)

以下で順番に対応を紹介していきます。

## テストの並列実行 - マトリックスの使用

GithubActions では、[matrix 構文](https://docs.github.com/ja/actions/using-jobs/using-a-matrix-for-your-jobs) を使うと変数の組み合わせによって、複数ジョブで並列実行できます。
以下のように設定すると、4 並列で実行。

```yaml
jobs:
  example:
    strategy:
      matrix: [0, 1, 2, 3]
    steps:
      ︙
```

今回のケースでは、develop ブランチとその他ブランチではコスト的な問題から並列ジョブ数を動的に変更したかったため以下のように設定しました。

```yaml
  set-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
    - name: Set matrix for strategy
      id: set-matrix
      run: |
        if [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
          echo "matrix={\"id\":[0,1,2,3,4],\"ruby\":[\"3.1\"],\"os\":[\"ubuntu-latest\"]}" >> $GITHUB_OUTPUT
        else
          echo "matrix={\"id\":[0,1,2],\"ruby\":[\"3.1\"],\"os\":[\"ubuntu-latest\"]}" >> $GITHUB_OUTPUT
        fi

  rspec:
    needs: [set-matrix]
    timeout-minutes: 20 # 1Jobにかかる大凡の見込み完了時間から設定、異常時に 6時間動き続けることを防止
    strategy:
      fail-fast: false # 失敗しても、他のJobがCancelされないよう
      matrix: ${{fromJson(needs.set-matrix.outputs.matrix)}}
    runs-on: ${{ matrix.os }}
    steps:
      ︙
```


## テストケースの分割 - 実行時間毎にテストケースを適切に分割

CircleCI であれば [テストの自動分割](https://circleci.com/docs/use-the-circleci-cli-to-split-tests/#split-by-timing-data) をサポートしてますが、GithubActions にはありません。
ただ、この仕組みを実現するカスタム Actions は、様々なものが公開されてます。
今回は r7kamura さんの [split-tests-by-timings](https://github.com/r7kamura/split-tests-by-timings) を使用させて貰いました。
以下のようにフローを作ることで、テスト結果時間毎にケースを適切に分割し実行できます。

```yaml
  rspec:
    strategy:
      fail-fast: false
      matrix: ${{fromJson(needs.set-matrix.outputs.matrix)}}
    runs-on: ${{ matrix.os }}
    steps:
      - name: download test report
        uses: dawidd6/action-download-artifact@v2
        with:
          branch: develop
          name: junit-xml-reports
          path: tmp/junit-xml-reports-downloaded
        continue-on-error: true

      - uses: r7kamura/split-tests-by-timings@v0
        id: split-tests
        with:
          reports: tmp/junit-xml-reports-downloaded
          glob: spec/**/*_spec.rb
          index: ${{ strategy.job-index }}
          total: ${{ strategy.job-total }}

      - run : |
        bundle exec rspec \
          --format progress \
          --format RspecJunitFormatter \
          --out tmp/junit-xml-reports/junit-xml-report-${{ strategy.job-index }}.xml \
          ${{ steps.split-tests.outputs.paths }}

      - if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v3
        with:
          if-no-files-found: error
          name: junit-xml-reports
          path: tmp/junit-xml-reports
```

# 対応後

![](https://storage.googleapis.com/zenn-user-upload/fec02dece79f-20231001.png)

実施した変更は非常にシンプルですが、この変更だけでもテスト時間が 38min -> 16min と半分以下に短縮されました。
実際の課金時間である Billable time も 40min -> 60min と 20min 少々余分にコストがかかる程度です。

コストの観点から考えると、標準のテストランナーを使用する場合、1 分あたり$0.008 かかります。
問題のあったリリース時に限ってみれば、このプロジェクトでは月に約 80 回のリリースが行われるため、$12 ほどの追加コストがかかりますが、待機時間が 1700 分も削減されることを考えると、それは十分に支払う価値があるラインでしょう。

また、ジョブの数を更に分割(30 個~)することで、 テストの実行時間を約 3~5 分に短縮できました。
しかし、分単位で切り上げて課金されるため、ノードを分割すればするほど 1 ノードあたり 1 分の追加課金が発生し、これはあえなく断念。
(sec 単位で課金されるとイイナー)


# まとめ

記事内のジョブは RSpec に特化していますが、JUnit XML format で結果が出力できれば、汎用的に使えます。
並列実行してないテストフローがあれば、少しの手間でテスト時間が大幅に改善されることもあるため、お試し下さい。

# 関連記事

https://zenn.dev/overflow_offers/articles/20230126-improve_rspec
https://zenn.dev/overflow_offers/articles/20221107-github_actions_frontend_build_test
