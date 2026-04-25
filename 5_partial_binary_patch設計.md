では **Phase5：partial binary patch 設計** に入る。
ここが最終目的に一番近い。

# Phase5のゴール

最終的に作りたいのはこれ：

```text
bufferの中の、指定されたフィールドだけを、安全に差し替える仕組み
```

つまり：

```text
buffer + layout + patch request
↓
指定範囲だけ更新
```

---

# 1. partial binary patch とは何か

直訳すると：

```text
バイナリデータの部分差し替え
```

あなたの現場問題に置き換えると：

```text
読み込んだ生データ buffer のうち、
馬主だけ、騎手だけ、日付だけ、など
一部フィールドだけを差し替える
```

ということ。

重要なのは **全部作り直さない** こと。

---

# 2. なぜ構造体丸ごと入れ替えが危険か

例えば Format A：

```text
offset 0  birthday
offset 4  owner
offset 8  jockey
```

Format B：

```text
offset 0  birthday
offset 4  jockey
offset 8  owner
```

このとき `owner` だけ変えたいのに、構造体ごとコピーすると：

```text
関係ない jockey や birthday まで壊す可能性がある
```

だから正しい考え方は：

```text
ownerのoffsetとsizeだけを見て、そこだけmemcpyする
```

---

# 3. 設計に必要な3要素

partial binary patchには最低限この3つが必要。

```text
① buffer
② layout
③ patch request
```

それぞれの意味：

```text
buffer        = 実データ
layout        = どこに何があるかの地図
patch request = 何を何に差し替えるか
```

---

# 4. 最小設計

まずフィールドID。

```cpp
enum class FieldID
{
    Birthday,
    Owner,
    Jockey
};
```

フィールド定義。

```cpp
struct FieldDescriptor
{
    size_t offset;  // buffer先頭から何バイト目か
    size_t size;    // 何バイト分か
};
```

layout。

```cpp
#include <unordered_map>

using LayoutTable = std::unordered_map<FieldID, FieldDescriptor>;
```

patch request。

```cpp
struct PatchRequest
{
    FieldID field;             // どのフィールドを差し替えるか
    const uint8_t* newData;    // 新しいバイト列
    size_t newDataSize;        // 新しいデータのサイズ
};
```

---

# 5. 最小patch関数

```cpp
#include <cstdint>
#include <cstddef>
#include <cstring>
#include <unordered_map>

enum class FieldID
{
    Birthday,
    Owner,
    Jockey
};

struct FieldDescriptor
{
    size_t offset;
    size_t size;
};

using LayoutTable = std::unordered_map<FieldID, FieldDescriptor>;

struct PatchRequest
{
    FieldID field;
    const uint8_t* newData;
    size_t newDataSize;
};

bool applyPatch(uint8_t* buffer,
                size_t bufferSize,
                const LayoutTable& layout,
                const PatchRequest& request)
{
    // 1. 指定フィールドがlayoutに存在するか確認する
    auto it = layout.find(request.field);
    if (it == layout.end())
    {
        return false;
    }

    const FieldDescriptor& desc = it->second;

    // 2. 書き換え範囲がbufferの外に出ないか確認する
    if (desc.offset + desc.size > bufferSize)
    {
        return false;
    }

    // 3. 固定長フィールドなのでサイズ一致を必須にする
    if (request.newDataSize != desc.size)
    {
        return false;
    }

    // 4. 指定範囲だけ差し替える
    std::memcpy(buffer + desc.offset,
                request.newData,
                desc.size);

    return true;
}
```

これが最小の partial binary patch。

---

# 6. 使用例

```cpp
#include <iostream>
#include <cstdint>
#include <cstddef>
#include <cstring>
#include <unordered_map>

enum class FieldID
{
    Birthday,
    Owner,
    Jockey
};

struct FieldDescriptor
{
    size_t offset;
    size_t size;
};

using LayoutTable = std::unordered_map<FieldID, FieldDescriptor>;

struct PatchRequest
{
    FieldID field;
    const uint8_t* newData;
    size_t newDataSize;
};

bool applyPatch(uint8_t* buffer,
                size_t bufferSize,
                const LayoutTable& layout,
                const PatchRequest& request)
{
    auto it = layout.find(request.field);
    if (it == layout.end())
    {
        return false;
    }

    const FieldDescriptor& desc = it->second;

    if (desc.offset + desc.size > bufferSize)
    {
        return false;
    }

    if (request.newDataSize != desc.size)
    {
        return false;
    }

    std::memcpy(buffer + desc.offset,
                request.newData,
                desc.size);

    return true;
}

void printBuffer(const uint8_t* buffer, size_t size)
{
    for (size_t i = 0; i < size; i++)
    {
        std::cout << (int)buffer[i] << " ";
    }
    std::cout << std::endl;
}

int main()
{
    uint8_t buffer[12] = {
        20, 24, 4, 25,
        1, 2, 3, 4,
        80, 81, 82, 83
    };

    LayoutTable layoutA = {
        { FieldID::Birthday, {0, 4} },
        { FieldID::Owner,    {4, 4} },
        { FieldID::Jockey,   {8, 4} }
    };

    uint8_t newOwner[4] = {9, 9, 9, 9};

    PatchRequest request{
        FieldID::Owner,
        newOwner,
        sizeof(newOwner)
    };

    bool ok = applyPatch(buffer,
                         sizeof(buffer),
                         layoutA,
                         request);

    std::cout << "patch result: " << ok << std::endl;
    printBuffer(buffer, sizeof(buffer));
}
```

結果：

```text
patch result: 1
20 24 4 25 9 9 9 9 80 81 82 83
```

ownerの4バイトだけ変わっている。

---

# 7. 設計として一番大事な思想

この設計の良いところは：

```text
patch処理は1個
layoutだけ差し替える
```

こと。

Format A：

```cpp
LayoutTable layoutA = {
    { FieldID::Birthday, {0, 4} },
    { FieldID::Owner,    {4, 4} },
    { FieldID::Jockey,   {8, 4} }
};
```

Format B：

```cpp
LayoutTable layoutB = {
    { FieldID::Birthday, {0, 4} },
    { FieldID::Jockey,   {4, 4} },
    { FieldID::Owner,    {8, 4} }
};
```

同じpatch関数でどちらも扱える。

```cpp
applyPatch(buffer, bufferSize, layoutA, request);
applyPatch(buffer, bufferSize, layoutB, request);
```

---

# 8. 現場設計ではさらに必要なもの

実務では最低限、次も入れる。

```text
① buffer範囲チェック
② field存在チェック
③ size一致チェック
④ nullチェック
⑤ format判定
⑥ ログ出力
⑦ パッチ前後の差分確認
```

特に危険なのはここ：

```cpp
std::memcpy(buffer + desc.offset, request.newData, desc.size);
```

この1行は強力だけど、間違えると普通にデータを壊す。

だから事前チェックが必須。

---

# 9. より実務寄りの関数

```cpp
bool applyPatchSafe(uint8_t* buffer,
                    size_t bufferSize,
                    const LayoutTable& layout,
                    const PatchRequest& request)
{
    // bufferがnullptrなら処理できない
    if (buffer == nullptr)
    {
        return false;
    }

    // 差し替え元データがnullptrなら処理できない
    if (request.newData == nullptr)
    {
        return false;
    }

    // 指定フィールドがlayoutにあるか確認する
    auto it = layout.find(request.field);
    if (it == layout.end())
    {
        return false;
    }

    const FieldDescriptor& desc = it->second;

    // offset + size がオーバーフローしないか確認する
    if (desc.offset > bufferSize)
    {
        return false;
    }

    // 書き換え範囲がbuffer内に収まるか確認する
    if (desc.size > bufferSize - desc.offset)
    {
        return false;
    }

    // 固定長フィールドなのでサイズ一致を必須にする
    if (request.newDataSize != desc.size)
    {
        return false;
    }

    // ここまで来て初めて書き換える
    std::memcpy(buffer + desc.offset,
                request.newData,
                desc.size);

    return true;
}
```

地味だけど、これが現場で信用されるコード。

---

# 10. 今回の問題を一文で説明するなら

リーダーに説明するなら、こう言えばよい。

```text
構造体全体を差し替えるのではなく、フォーマットごとのlayout descriptorを用意し、
field IDからoffset/sizeを引いて、該当範囲だけをmemcpyでpatchする設計にします。
これにより、レイアウト差異をpatch処理から分離できます。
```

これが言えれば、理解している人に見える。

---

# Phase5の結論

今回の問題の正体はこれ：

```text
bufferを直接書き換える問題ではなく、
layout差異を吸収しながら安全に部分更新する設計問題
```

だから設計の中心は：

```text
structではなくlayout descriptor
```

そして処理の中心は：

```text
offset + size + memcpy
```

です。
