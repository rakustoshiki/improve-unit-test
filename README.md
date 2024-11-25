# Vitest 実行時間改善記録

## 定義

| 定義 |  |
| ---- | ---- |
| ユニットテスト | コンポーネントのrenderを必要としないテスト<br>拡張子がts |
| コンポーネントテスト | コンポーネントのrenderを必要とするテスト<br>拡張子がtsx |

| 指標            | 説明                                                          |
| --------------- | ------------------------------------------------------------- |
| **Duration**    | テスト全体の実行時間。以下の各フェーズの合計時間です。        |
| **Transform**   | テストファイルやその依存関係の変換に要した時間。              |
| **Setup**       | `setupFiles`で指定されたファイルの実行に要した時間。          |
| **Collect**     | テストファイル内のテストケースを収集するための時間。          |
| **Tests**       | 実際のテストケースの実行に要した時間。                        |
| **Environment** | テスト環境のセットアップに要した時間（例：JSDOMの設定など）。 |
| **Prepare**     | テストランナーの準備に要した時間。                            |


## 前提

- キャッシュなしの状態で測定する
  - `vite.config`で`cacheDir: "node_modules/.test"`を設定し毎回削除
- Vitestのバージョンは`v2.1.5`
## ユニットテスト

### ベンチマーク

```ts
const sum = (a: number, b: number) => a + b;

describe("example", () => {
  it("example", () => {
    expect(sum(1, 2)).toEqual(3);
  });
});
```

### 改善前の状態
sum: 1.95s 
| 項目          | 時間      |
|---------------|-----------|
| Transform     | 55ms      |
| Setup         | 368ms     |
| Collect       | 6ms       |
| Tests         | 3ms       |
| Environment   | 1.01s     |
| Prepare       | 185ms     |

### 改善１: aaa

## コンポーネントテスト

### ベンチマーク

```tsx
import { render, screen } from "@testing-library/react";
import { TestProviders } from "tests/TestProviders";

import { KanjoKamokuSettingsListContainer } from "src/components/features/kanjoKamokuSettings/container/KanjoKamokuSettingsListContainer";
import type { KanjoKamokuListResponse } from "src/components/features/kanjoKamokuSettings/kanjoKamokuSettings.type";

describe("example", () => {
  const data: KanjoKamokuListResponse = {
    kanjoKamokuLimit: 1000,
    pagination: {
      page: 1,
      totalPages: 1,
      total: 1,
    },
    resultList: [
      {
        kanjoKamokuId: 1,
        code: "kamoku001",
        kanjoKamokuName: "普通預金",
        memo: "長いメモ０１２３４５６７８９ＡＢＣＤＥＦＧＨＩＪＫＬＭＮ",
        version: 1,
        isDeletable: true,
      },
      {
        kanjoKamokuId: 2,
        code: "kamoku002",
        kanjoKamokuName: "当座預金",
        memo: "短いメモ",
        version: 1,
        isDeletable: false,
      },
    ],
  };

  it("データが正常に表示されること", () => {
    // arrange
    render(<KanjoKamokuSettingsListContainer kanjoKamokuListData={data} />, {
      wrapper: TestProviders,
    });

    // assert
    const rows = screen.getAllByRole("row");
    const [checkbox, editButton, code, kanjoKamokuName, memo] =
      rows[1].children;

    expect(checkbox).toBeInTheDocument();
    expect(editButton).toBeInTheDocument();
    expect(code.textContent).toEqual("kamoku001");
    expect(kanjoKamokuName.textContent).toEqual("普通預金");
    expect(memo.textContent).toEqual(
      "長いメモ０１２３４５６７８９ＡＢＣＤＥＦＧＨＩＪＫＬＭＮ"
    );
  });
});
```

### 改善前の状態
sum: 11.55s
| 項目          | 時間      |
|---------------|-----------|
| Transform     | 810ms     |
| Setup         | 290ms     |
| Collect       | 9.42s     |
| Tests         | 371ms     |
| Environment   | 1.04s     |
| Prepare       | 113ms     |

- ほぼ`Collect`の時間がテストの時間を決めている
- Collectは計測のブレが大きい
  - 6~13s程度のブレがある
  - キャッシュは関係ない

### 命題1: **Collect**が大きいのはなぜ？
1. 読み込んでいるコンポーネントのファイルサイズが大きい？

### 改善1: ベンチマークを軽いコンポーネントに変えてみる

```ts
export const TestContainer = () => {
  return <div>TestContainer</div>;
};
```

sum: 2.17s
| 項目          | 時間      |
|---------------|-----------|
| Transform     | 53ms      |
| Setup         | 176ms     |
| Collect       | 141ms     |
| Tests         | 45ms      |
| Environment   | 426ms     |
| Prepare       | 80ms      |

- かなり改善
  - ボトルネック=コンポーネントファイルサイズとみて良さそう
- 計測値のブレも少ない2~2.2秒
- じゃあ読み込むコンポーネントサイズを削減すればいい→一概にはできない
  - mockは必ず必要
  - Jotaiとかstyled-componentを使っているのでProviderで必ずラップしたい
  - ファイルを分けるなら実装時からテストのことを意識した設計にしなければいけない
- **ファイルサイズを変えず`Collect`の数値を改善できないか**
- **Collectを改善せず全体のテスト時間を改善できないか**

### 命題2: Collectを改善せず全体のテスト時間を改善できないか
- Vitest公式Docに[Improving Performance](https://vitest.dev/guide/improving-performance)の記述がある
- こちらはCollectの改善というより、全体のテスト時間削減について書かれている
  - 一通り試してみる

以下ディレクトリのテスト全てが終了するまでの時間を検証する

```text
tests/features/billingDetail
├── billingDetail.spec.tsx
├── billingInfoEditValidator.spec.tsx
└── memoEditValidator.spec.tsx
```

sum: 15.91s
| 項目         | 時間     |
|--------------|----------|
| Transform    | 788ms    |
| Setup        | 612ms    |
| Collect      | 28.67s   |
| Tests        | 6.47s    |
| Environment  | 1.73s    |
| Prepare      | 280ms    |

### 改善2-1: `--no-isolate`フラグをつけて実行

sum: 16.15s
| 項目         | 時間     |
|--------------|----------|
| Transform    | 958ms    |
| Setup        | 551ms    |
| Collect      | 28.80s   |
| Tests        | 6.58s    |
| Environment  | 1.47s    |
| Prepare      | 256ms    |

- 特に変化なし

### 改善2-2: `--no-file-parallelism`フラグをつけて実行

sum: 49.37s
| 項目         | 時間     |
|--------------|----------|
| Transform    | 720ms    |
| Setup        | 1.56s    |
| Collect      | 33.57s   |
| Tests        | 6.19s    |
| Environment  | 3.02s    |
| Prepare      | 276ms    |

- かなり遅くなる

### 改善2-3: poolをthredsにして実行（デフォルトはforks）

sum: 12.06s
| 項目         | 時間     |
|--------------|----------|
| Transform    | 916ms    |
| Setup        | 650ms    |
| Collect      | 23.82s   |
| Tests        | 6.26s    |
| Environment  | 1.73s    |
| Prepare      | 253ms    |

- 微改善

### 改善2-4: shardingを試す
https://vitest.dev/guide/improving-performance#sharding
TBD

### 雑記
`yarn test tests/features/shiwakeExportSettings/shiwakeExportSettings.spec.tsx`意外にも早い
Duration  15.50s (transform 865ms, setup 170ms, collect 7.31s, tests 6.16s, environment 438ms, prepare 84ms)
むしろ全部モックした方がはやい？

ベンチマーク以下に変えてみたけど、あんま変わらんかった

```ts
import { render, screen } from "@testing-library/react";
import { tanStackQueryClient } from "src/lib/tanStackQueryClient";
import { TestProviders } from "tests/TestProviders";
import { server } from "tests/mocks/server";

import KanjoKamokuSettingsContainer from "src/components/pages/KanjoKamokuSettings";

describe("example", () => {
  beforeAll(() => {
    server.listen({ onUnhandledRequest: "error" });
  });

  afterEach(() => {
    tanStackQueryClient.clear();
    server.resetHandlers();
  });

  afterAll(() => {
    server.close();
  });

  it("データが正常に表示されること", async () => {
    // arrange
    render(<KanjoKamokuSettingsContainer />, {
      wrapper: TestProviders,
    });
    await screen.findByText("勘定科目設定");

    // assert
    const rows = screen.getAllByRole("row");
    const [checkbox, editButton, code, kanjoKamokuName, memo] =
      rows[1].children;

    expect(checkbox).toBeInTheDocument();
    expect(editButton).toBeInTheDocument();
    expect(code.textContent).toEqual("kanjo001");
    expect(kanjoKamokuName.textContent).toEqual("普通預金");
    expect(memo.textContent).toEqual("メモ");
  });
});
```

sum: 11.34s
| 項目         | 時間     |
|--------------|----------|
| Transform    | 924ms    |
| Setup        | 259ms    |
| Collect      | 8.51s    |
| Tests        | 574ms    |
| Environment  | 543ms    |
| Prepare      | 91ms     |

目標：157s以下
