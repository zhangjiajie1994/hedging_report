# hedging_report
hedging_report

可以。既然你最后统一使用 **Gemini 3.5 Flash**，我建议把整个方案定成一个非常简单的版本：

> **PDF解析 → 分块事实提取 → 全部事实合并 → Gemini总摘要 → 货币覆盖审查 → 缺失时定向回查 → 重新生成最终摘要**

核心原则只有两个：

> **前面不做“摘要”，只做事实提取；最后只做一次真正的摘要。**  
> **审查不是重新读150页，而是先程序化找“漏掉的货币”，有异常才回查对应页面。**

这样对你现在系统的改动最少。

---

# 一、最终推荐架构

```text
                    150页 PDF
                        │
                        ▼
              ① PDF解析 + 保留页码
                        │
                        ▼
              ② 10–15页组成Chunk
                 相邻重叠1–2页
                        │
                        ▼
               ③ Gemini 3.5 Flash
                   分块事实提取
                        │
                        ▼
              Chunk事实1
              Chunk事实2
              Chunk事实3
                   ......
                        │
                        ▼
               ④ 合并全部事实
                  去重、标准化
                        │
                        ▼
                 全局事实池
                        │
                        ▼
               ⑤ Gemini 3.5 Flash
                   一次性总摘要
                        │
                        ▼
                  Summary V1
                        │
                        ▼
            ⑥ Currency Coverage Audit
               PDF货币集合
                    VS
               事实池货币集合
                        │
            ┌───────────┴───────────┐
            │                       │
          全覆盖                    有遗漏
            │                       │
           PASS              找出缺失货币页码
                                    │
                                    ▼
                          ⑦ 只回查相关Chunk
                                    │
                                    ▼
                          Gemini重新提取事实
                                    │
                                    ▼
                            补入全局事实池
                                    │
                                    ▼
                          ⑧ 重新生成最终摘要
                                    │
                                    ▼
                               Summary V2
```

这就是我最建议你交给leader的版本。

---

# 二、每一步具体怎么做，以及为什么

1. **PDF解析：推荐 `PyMuPDF4LLM` 为主，`pdfplumber`作为表格兜底。** 首先不要直接把PDF交给Gemini，而是先转成带页码的Markdown或文本。主流程可以用 `PyMuPDF4LLM`，因为你后面本来就是给LLM处理，它输出Markdown比较方便，而且容易保留页级信息。如果发现财务表格解析明显有问题，再只对这些页面调用 `pdfplumber` 提取表格，而不是一开始就把系统搞复杂。这样做的原因是，你真正需要的是“哪条数据来自哪一页”，后面的审查机制完全依赖页码。如果PDF解析阶段把页码丢了，后面即使发现JPY漏了，也不知道应该回查哪里。建议最基本的数据结构保留成：
   
   ```python
   pages = [
       {
           "page": 1,
           "text": "...",
           "tables": [...]
       },
       {
           "page": 2,
           "text": "...",
           "tables": [...]
       }
   ]
   ```

2. **分块：建议10–15页一个Chunk，相邻重叠1–2页。** 比如 `P1–15 → P14–28 → P27–41`。不要一页一页送，也不要150页一次性送。财报里一个表格经常从P37跨到P38，如果按照单页处理，P37只有表头，P38只有数字，模型很容易理解错误；保留1–2页重叠后，同一个表格大概率至少在一个Chunk中是完整的。15页左右对Gemini 3.5 Flash也比较容易稳定处理。你还可以在切块时优先按章节切，如果P15正好是一个大表格中间，可以把这个Chunk延长到P16或P17，不必死守15页。

3. **Gemini分块阶段只做“事实提取”，不要让它摘要。** 这是整个方案最关键的一点。例如不要让它输出“SGD业务总体有所增长”，而应该输出：
   
   ```text
   Entity: Singapore business
   Currency: SGD
   Year: 2024
   Metric: Revenue
   Value: 91
   Unit: million
   Page: 37
   
   Entity: Singapore business
   Currency: SGD
   Year: 2025
   Metric: Revenue
   Value: 105
   Unit: million
   Page: 38
   ```
   
   原因是“摘要”本身就是信息压缩。如果150页先被压成10个局部摘要，再把10个摘要压成一个总摘要，相当于连续两次有损压缩，数字、年份、小币种和表格中的次要字段最容易消失。所以流程应该是：
   
   ```text
   PDF
   → Facts
   → Facts
   → Facts
   → 最后才Summary
   ```
   
   而不是：
   
   ```text
   PDF
   → Summary
   → Summary
   → Summary
   ```
   
   Gemini这一阶段的提示词建议固定要求它提取 `Entity / Currency / Year / Metric / Value / Unit / Period / Page / Notes`，没有的信息填null，不允许猜测。

4. **合并事实时不要使用LLM重新总结，而是程序化合并和去重。** 所有Chunk跑完以后，把它们合并成一个全局 `fact_pool`。例如相邻Chunk因为有重叠，P14的同一条USD数据可能出现两次，这时候根据 `Page + Currency + Year + Metric + Value` 去重即可。同时统一币种写法，例如：
   
   ```text
   US$
   USD
   U.S. dollars
   US dollars
   ```
   
   全部标准化成：
   
   ```text
   USD
   ```
   
   `S$ / SGD / Singapore dollars`统一成 `SGD`。这样做是为了避免后面的审查误判。否则PDF写的是“US$”，事实池写的是“USD”，程序可能错误地认为USD没被提取。

5. **只有到了这里，才让Gemini 3.5 Flash生成一次完整摘要。** 输入不再是150页PDF，而是已经结构化、去重后的完整事实池。比如系统已经得到USD、SGD、EUR各年度数据，就让Gemini基于完整事实做趋势、同比、跨年度和币种比较。提示词里强调：
   
   ```text
   只能依据提供的事实池生成摘要。
   不得补充事实池中不存在的数据。
   对同一币种应结合不同年份进行纵向比较。
   对不同币种/业务主体在可比情况下进行横向比较。
   保留重要数字、转折点和异常变化。
   ```
   
   为什么这样做？因为这时候Gemini看到的是“经过清洗的完整证据”，它只负责它最擅长的事情：组织、比较和语言生成，而不是同时承担PDF解析、表格识别、信息检索和摘要四种任务。

6. **最后增加一个非常轻的“货币覆盖审查”，作为leader要求的审查机制。** 这一步我建议首先**不用Gemini**，直接Python程序做。PDF解析的时候扫描所有货币实体，得到：
   
   ```python
   pdf_currency_pages = {
       "USD": [12, 13, 58, 59],
       "SGD": [21, 22, 36],
       "EUR": [47, 48],
       "JPY": [87, 88]
   }
   ```
   
   然后事实池统计：
   
   ```python
   fact_currencies = {
       "USD",
       "SGD",
       "EUR"
   }
   ```
   
   一比较：
   
   ```python
   missing_currencies = (
       set(pdf_currency_pages.keys())
       - fact_currencies
   )
   ```
   
   得到：
   
   ```python
   {"JPY"}
   ```
   
   这时候系统就知道：**JPY在源PDF里出现了，但事实提取阶段没有进入事实池。** 这比让Gemini重新读150页然后问“有没有遗漏？”成本低得多，而且判断非常明确。

7. **发现货币缺失后，不直接把它写进最终摘要，而是“定向回查”。** 例如程序知道JPY出现在P87、P88，就取：
   
   ```text
   P86–P89
   ```
   
   或者找到P87所在的原始Chunk，重新交给Gemini。提示词只问：
   
   ```text
   PDF中检测到JPY，但当前事实池没有JPY相关事实。
   
   请检查以下页面。
   
   判断JPY是否包含与当前任务相关的重要财务/年报信息。
   
   如果重要：
   按 Entity / Currency / Year / Metric /
   Value / Unit / Period / Page / Notes
   输出事实。
   
   如果JPY仅出现在：
   - 示例
   - 脚注
   - 通用汇率列表
   - 无关文字
   - 目录
   
   则输出 IGNORE。
   
   不要重新总结页面。
   ```
   
   这一步很重要，因为“PDF出现JPY”并不等于“摘要必须包含JPY”。可能只是脚注写了一句“USD/JPY exchange rate”。所以程序负责**发现风险**，Gemini负责**判断是否真的需要补充**。

8. **如果确定确实遗漏，把新事实加入事实池，然后重新生成摘要，不要直接在旧摘要上打补丁。** 比如原事实池漏掉：
   
   ```text
   JPY
   2024: xxx
   2025: xxx
   ```
   
   那就补回：
   
   ```text
   Fact Pool V1
   +
   JPY missing facts
   =
   Fact Pool V2
   ```
   
   再让Gemini根据 `Fact Pool V2` 重新生成Summary V2。为什么不让它“在原摘要里补一句JPY”？因为补丁式修改做多以后，很容易出现前后矛盾，例如前面写“三种主要货币”，后面又突然出现第四种。重新基于完整事实池生成，整体逻辑最干净。

---

# 三、货币审查不要只写一个简单正则

这是实现的时候容易踩坑的地方。

你不能只搜：

```python
USD|SGD|EUR|JPY
```

因为年报里可能写：

```text
US$
USD
U.S. dollars
US dollars
$

S$
SGD
Singapore dollars

RMB
CNY
Renminbi
人民币

JPY
Japanese yen
¥
```

所以建议维护一个很小的alias dictionary：

```python
CURRENCY_ALIASES = {
    "USD": [
        "USD",
        "US$",
        "U.S. dollar",
        "U.S. dollars",
        "US dollar",
        "US dollars"
    ],
    "SGD": [
        "SGD",
        "S$",
        "Singapore dollar",
        "Singapore dollars"
    ],
    "EUR": [
        "EUR",
        "euro",
        "euros"
    ],
    "JPY": [
        "JPY",
        "Japanese yen"
    ],
    "CNY": [
        "CNY",
        "RMB",
        "Renminbi"
    ]
}
```

我反而不建议一开始单独依赖 `$`、`¥` 这种符号，因为歧义比较大。

例如 `$100 million` 到底USD还是SGD，要结合报告上下文判断。

---

# 四、我建议审查“事实池”，不要只审查最终摘要

这个细节很重要。

你的审查关系应该是：

```text
PDF currencies
       ↓
      VS
       ↓
Fact Pool currencies
```

而不是：

```text
PDF currencies
       ↓
      VS
       ↓
Final Summary currencies
```

因为最终摘要本来就允许压缩。

举个例子：

PDF和事实池都有：

```text
USD
SGD
EUR
JPY
```

但是Gemini最终摘要只提：

```text
USD、SGD、EUR
```

不一定是错误。

可能JPY只有一条次要数据，不值得进入高级摘要。

所以：

> **PDF → Fact Pool** 检查的是“信息有没有被系统漏掉”。

而：

> **Fact Pool → Summary** 是“哪些信息值得呈现”的摘要策略问题。

这两个不能混在一起。

因此你们leader所说的“审查机制”，我建议正式定义为：

> **Source-to-Fact Coverage Audit（源文档—事实池覆盖审查）**

比“检查最终摘要有没有每个币种”更合理。

---

# 五、如果你想再加一点点保险，但又不增加多少工作量

你现在检查：

```text
Currency
```

已经可以了。

如果代码允许，我建议升级成：

```text
Currency + Year
```

例如PDF里存在：

```text
USD 2022
USD 2023
USD 2024
USD 2025
```

事实池里只有：

```text
USD 2023
USD 2024
USD 2025
```

只做currency检查会认为：

```text
USD存在
→ PASS
```

但其实2022年整年被漏了。

如果做：

```python
("USD", 2022)
("USD", 2023)
("USD", 2024)
("USD", 2025)
```

就能发现：

```text
Missing:
USD + 2022
```

代码不会复杂多少，但可靠性会明显更高。

所以我建议你的审查最低版本：

> **Currency Coverage**

稍微增强版：

> **Currency-Year Coverage**

如果是我来定项目，我会直接用后者。

---

# 六、最后你可以把整个系统概括成四层

这是你给leader汇报时最好说的：

```text
第一层：文档解析
PDF → 带页码的文本、表格

第二层：信息获取
Chunk → Gemini 3.5 Flash → 结构化事实

第三层：信息生成
事实合并 → Gemini 3.5 Flash → 全文摘要

第四层：质量审查
源PDF Currency-Year
        VS
Fact Pool Currency-Year

一致 → PASS

不一致
→ 定位源页
→ Gemini定向回查
→ 补充事实
→ 重新生成摘要
```

一句话版本可以写成：

> **采用“分块事实提取—全局事实聚合—一次性摘要生成—源文档覆盖回查”的处理流程，并以Currency-Year实体覆盖作为轻量级质量审查机制；仅当发现源PDF实体未进入事实池时触发定向回溯与重新提取，从而在较低额外Token消耗下减少长PDF和跨页表格导致的信息遗漏。**

我认为这已经是一个**很适合你现在实际落地的方案**：不用RAG、不需要向量数据库、不需要多Agent，也不用改变你已经选定的 **Gemini 3.5 Flash**，主要就是增加 **结构化事实池 + Currency-Year审查 + 定向回查** 三个很轻的模块。