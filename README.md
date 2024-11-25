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

### 改善1: 
