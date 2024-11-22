# Vitest 実行時間改善記録

## 定義

| 定義 |  |
| ---- | ---- |
| ユニットテスト | コンポーネントのrenderを必要としないテスト<br>拡張子がts |
| コンポーネントテスト | コンポーネントのrenderを必要とするテスト<br>拡張子がtsx |

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
sum: 13.34s 
| 項目          | 時間      |
|---------------|-----------|
| Transform     | 1.05s     |
| Setup         | 201ms     |
| Collect       | 11.38s    |
| Tests         | 517ms     |
| Environment   | 655ms     |
| Prepare       | 272ms     |
