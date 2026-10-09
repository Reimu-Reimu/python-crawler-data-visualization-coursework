# 基于 Python 与 AI 辅助开发的图书信息智能爬虫系统

> 使用 Python、Requests、BeautifulSoup 和 Matplotlib，实现图书信息自动采集、数据清洗、CSV 导出与可视化分析。

- **项目类型：** Python 网络爬虫与数据分析
- **开发语言：** Python 3
- **开发工具：** VS Code、GitHub、AI 辅助编程工具
- **核心技术：** Requests、BeautifulSoup、Pandas、Matplotlib
- **数据来源：** [Books to Scrape](https://books.toscrape.com/)，专门用于练习网页抓取的公开测试网站

---

## 一、项目背景

随着互联网的发展，网站中包含了大量有价值的信息。如果通过人工方式逐条收集，不仅效率较低，而且容易出现遗漏和重复。

网络爬虫是一种能够自动访问网页、提取指定信息并保存数据的程序。通过爬虫技术，可以将网页中分散的信息转换为结构化数据，为后续的数据分析提供基础。

本项目借助 AI 辅助编程工具，开发一个图书信息采集系统。系统能够自动访问图书网站，提取图书名称、价格、评分、库存状态和商品链接等信息，并将结果保存为 CSV 文件。

在完成数据采集后，程序还会对采集结果进行统计分析，生成图书评分分布图和价格分布图，使爬虫不仅具有数据采集能力，还具备初步的数据分析能力。

## 二、项目功能

本项目主要实现以下功能：

1. **自动访问网页：** 使用 Requests 发送 HTTP 请求，获取网页 HTML 内容。
2. **解析网页数据：** 使用 BeautifulSoup 提取图书名称、价格、评分和库存信息。
3. **自动翻页：** 根据网页中的下一页链接，连续采集多页数据。
4. **数据清洗：** 统一价格、评分和库存字段的格式，过滤不完整的数据。
5. **数据保存：** 使用 Pandas 将采集结果保存为 CSV 文件。
6. **数据可视化：** 使用 Matplotlib 生成价格分布图和评分分布图。
7. **异常处理：** 对网络请求失败、网页解析异常等情况进行处理。

## 三、开发环境

建议使用以下环境：

| 项目 | 配置 |
|---|---|
| 操作系统 | Windows 10/11 或 Linux |
| Python | Python 3.10 及以上 |
| 编辑器 | Visual Studio Code |
| HTTP 请求库 | Requests |
| HTML 解析库 | BeautifulSoup4 |
| 数据处理库 | Pandas |
| 数据可视化库 | Matplotlib |

安装所需依赖：

```bash
pip install requests beautifulsoup4 pandas matplotlib
```

## 四、爬虫程序设计

本项目采用模块化设计，将程序划分为网页请求、数据解析、数据保存和数据分析四个部分。

整体工作流程如下：

1. 访问图书网站首页。
2. 获取网页 HTML 源代码。
3. 解析网页中的图书信息。
4. 提取图书名称、价格、评分和库存状态。
5. 判断是否存在下一页，并继续采集。
6. 对采集结果进行去重和清洗。
7. 将数据保存到 CSV 文件。
8. 生成数据分析图表。

通过这种设计，可以使程序结构更加清晰，也便于后续扩展其他采集功能。

## 五、完整源代码

创建文件 `crawler.py`，写入以下代码。

```python
import time
from pathlib import Path
from urllib.parse import urljoin

import requests
import pandas as pd
import matplotlib.pyplot as plt
from bs4 import BeautifulSoup


# 网站地址：专门用于练习网页抓取的测试网站
BASE_URL = "https://books.toscrape.com/"
OUTPUT_DIR = Path("results")

# 创建结果目录
OUTPUT_DIR.mkdir(exist_ok=True)

# 设置请求头，标识程序用途
HEADERS = {
    "User-Agent": "StudentBookCrawler/1.0 (educational project)"
}

# 网站评分文字与数字的对应关系
RATING_MAP = {
    "One": 1,
    "Two": 2,
    "Three": 3,
    "Four": 4,
    "Five": 5
}


def fetch_page(url, session):
    """请求网页并返回 HTML 内容。"""
    try:
        response = session.get(
            url,
            headers=HEADERS,
            timeout=15
        )
        response.raise_for_status()
        response.encoding = response.apparent_encoding
        return response.text

    except requests.RequestException as exc:
        print(f"网页请求失败：{url}")
        print(f"错误信息：{exc}")
        return None


def parse_page(html, page_url):
    """解析网页，提取当前页面的图书信息。"""
    soup = BeautifulSoup(html, "html.parser")
    books = []

    for item in soup.select("article.product_pod"):
        title_tag = item.select_one("h3 a")
        price_tag = item.select_one(".price_color")
        stock_tag = item.select_one(".availability")
        rating_tag = item.select_one(".star-rating")

        # 必要字段缺失时跳过当前记录
        if not title_tag or not price_tag:
            continue

        title = title_tag.get("title", "").strip()
        book_url = urljoin(
            page_url,
            title_tag.get("href", "")
        )

        # 将价格文本转换为浮点数
        price_text = price_tag.get_text(strip=True)
        price_text = price_text.replace("£", "")
        price = float(price_text)

        # 提取评分
        rating = None
        if rating_tag:
            classes = rating_tag.get("class", [])

            for class_name in classes:
                if class_name in RATING_MAP:
                    rating = RATING_MAP[class_name]
                    break

        # 提取库存状态
        stock = (
            stock_tag.get_text(" ", strip=True)
            if stock_tag
            else "未知"
        )

        books.append({
            "title": title,
            "price": price,
            "rating": rating,
            "stock": stock,
            "url": book_url
        })

    # 查找下一页链接
    next_tag = soup.select_one("li.next a")
    next_url = None

    if next_tag:
        next_url = urljoin(
            page_url,
            next_tag.get("href", "")
        )

    return books, next_url


def crawl_books(max_pages=3):
    """按页面顺序采集图书信息。"""
    all_books = []
    current_url = BASE_URL

    with requests.Session() as session:
        for page_number in range(1, max_pages + 1):
            if not current_url:
                print("没有下一页，采集结束。")
                break

            print(f"正在采集第 {page_number} 页：{current_url}")

            html = fetch_page(current_url, session)

            if html is None:
                print("当前页面请求失败，停止采集。")
                break

            try:
                books, next_url = parse_page(
                    html,
                    current_url
                )

            except (ValueError, AttributeError) as exc:
                print(f"网页解析失败：{exc}")
                break

            all_books.extend(books)
            print(f"本页采集到 {len(books)} 本图书")

            current_url = next_url

            # 控制请求频率，避免对网站造成压力
            if current_url and page_number < max_pages:
                time.sleep(1.5)

    return all_books


def save_data(books):
    """清洗数据并保存为 CSV 文件。"""
    if not books:
        print("没有采集到数据，无法保存。")
        return None

    df = pd.DataFrame(books)

    # 删除标题为空的记录
    df = df.dropna(subset=["title"])
    df = df[df["title"].str.strip() != ""]

    # 根据商品链接去重
    df = df.drop_duplicates(subset=["url"])

    # 保存为 UTF-8 with BOM，方便 Excel 打开中文数据
    output_file = OUTPUT_DIR / "books.csv"
    df.to_csv(
        output_file,
        index=False,
        encoding="utf-8-sig"
    )

    print(f"数据已保存：{output_file.resolve()}")
    print(f"有效图书数量：{len(df)}")

    return df


def analyze_data(df):
    """统计图书价格与评分并生成图表。"""
    if df is None or df.empty:
        print("数据为空，跳过分析。")
        return

    print("\n===== 图书数据统计 =====")
    print(f"图书总数：{len(df)}")
    print(f"平均价格：£{df['price'].mean():.2f}")
    print(f"最低价格：£{df['price'].min():.2f}")
    print(f"最高价格：£{df['price'].max():.2f}")

    valid_ratings = df["rating"].dropna()

    if not valid_ratings.empty:
        print(f"平均评分：{valid_ratings.mean():.2f}")

    # 设置图表字体，尽可能兼容中文环境
    plt.rcParams["font.sans-serif"] = [
        "Microsoft YaHei",
        "SimHei",
        "DejaVu Sans"
    ]
    plt.rcParams["axes.unicode_minus"] = False

    # 图表一：价格分布
    plt.figure(figsize=(8, 5))
    plt.hist(df["price"], bins=10, edgecolor="black")
    plt.title("Book Price Distribution")
    plt.xlabel("Price (£)")
    plt.ylabel("Number of Books")
    plt.tight_layout()

    price_chart = OUTPUT_DIR / "price_distribution.png"
    plt.savefig(price_chart, dpi=150)
    plt.close()

    # 图表二：评分分布
    if not valid_ratings.empty:
        rating_counts = (
            valid_ratings.astype(int)
            .value_counts()
            .reindex([1, 2, 3, 4, 5], fill_value=0)
            .sort_index()
        )

        plt.figure(figsize=(8, 5))
        rating_counts.plot(kind="bar")
        plt.title("Book Rating Distribution")
        plt.xlabel("Rating")
        plt.ylabel("Number of Books")
        plt.xticks(rotation=0)
        plt.tight_layout()

        rating_chart = OUTPUT_DIR / "rating_distribution.png"
        plt.savefig(rating_chart, dpi=150)
        plt.close()

    print("数据分析图表已生成。")


def main():
    print("===== 图书信息爬虫启动 =====")

    books = crawl_books(max_pages=3)
    df = save_data(books)
    analyze_data(df)

    print("===== 爬虫任务结束 =====")


if __name__ == "__main__":
    main()
```

## 六、核心代码解析

### 1. 使用 Requests 获取网页

```python
response = session.get(
    url,
    headers=HEADERS,
    timeout=15
)
response.raise_for_status()
```

Requests 负责向目标网站发送 HTTP 请求。

其中，`headers` 用于提供客户端标识等请求信息，`timeout` 用于限制请求等待时间，`raise_for_status()` 则能够在 HTTP 请求失败时触发异常。

程序通过异常处理避免因网络问题直接崩溃。

### 2. 使用 BeautifulSoup 解析 HTML

```python
soup = BeautifulSoup(html, "html.parser")

for item in soup.select("article.product_pod"):
    title_tag = item.select_one("h3 a")
    price_tag = item.select_one(".price_color")
```

BeautifulSoup 可以将 HTML 文档转换为便于查询的对象。

`select()` 和 `select_one()` 支持 CSS 选择器，可以定位网页中的指定元素。

本项目通过 `article.product_pod` 找到图书商品区域，再从中提取名称和价格。

这种方法比直接使用字符串截取更加灵活。

### 3. 自动翻页

```python
next_tag = soup.select_one("li.next a")

if next_tag:
    next_url = urljoin(
        page_url,
        next_tag.get("href", "")
    )
```

图书网站采用分页形式展示商品。

程序通过查找下一页链接获取后续页面地址，并使用 `urljoin()` 将相对地址转换为完整 URL。

在采集函数中，只要存在下一页且尚未达到指定页数，程序就会继续采集。

本项目默认采集三页，方便快速验证程序是否正常运行。

### 4. 数据清洗与保存

```python
df = pd.DataFrame(books)

df = df.dropna(subset=["title"])
df = df[df["title"].str.strip() != ""]
df = df.drop_duplicates(subset=["url"])

df.to_csv(
    output_file,
    index=False,
    encoding="utf-8-sig"
)
```

采集得到的数据可能存在字段缺失或重复记录，因此需要进行清洗。

程序首先删除标题为空的记录，再根据图书链接去除重复数据，最后将结果保存为 CSV 文件。

CSV 格式便于使用 Excel 打开，也可以继续用于 Python 数据分析。

### 5. 数据分析与可视化

```python
print(f"平均价格：£{df['price'].mean():.2f}")
print(f"最低价格：£{df['price'].min():.2f}")
print(f"最高价格：£{df['price'].max():.2f}")
```

Pandas 可以快速计算价格的平均值、最大值和最小值。

程序还使用 Matplotlib 生成价格分布图和评分分布图，使采集结果更加直观。

通过可视化结果，可以初步了解图书价格的分布情况，以及不同评分等级的图书数量。

## 七、程序运行方法

### 1. 创建项目目录

```text
python-book-crawler/
├── crawler.py
├── requirements.txt
├── README.md
└── results/
```

### 2. 创建依赖文件

在 `requirements.txt` 中写入：

```text
requests
beautifulsoup4
pandas
matplotlib
```

### 3. 安装依赖

```bash
python -m pip install -r requirements.txt
```

### 4. 运行爬虫

```bash
python crawler.py
```

如果网络正常，程序会依次请求图书网站的多个页面，打印每页采集到的图书数量，并生成 CSV 文件和分析图表。

运行结束后，项目目录中的 `results` 文件夹将包含：

```text
results/
├── books.csv
├── price_distribution.png
└── rating_distribution.png
```

## 八、实验结果与分析

### 1. 实验目的

验证程序能否正常访问测试网站、提取图书信息、自动翻页、保存数据并完成基本的数据分析。

### 2. 实验方法

设置最大采集页数为 3，每次请求之间等待 1.5 秒。

程序采集图书名称、价格、评分、库存状态和商品链接等字段，并将有效数据写入 CSV 文件。

### 3. 实验结果记录

程序运行后，可以根据实际控制台输出填写实验记录。

| 实验项目 | 实际结果 |
|---|---|
| 网页请求 | 以实际运行结果为准 |
| 成功采集页数 | 以实际运行结果为准 |
| 采集图书数量 | 以实际运行结果为准 |
| 数据清洗 | 检查空标题和重复链接的处理结果 |
| CSV 文件生成 | 检查 `results/books.csv` |
| 价格分析图 | 检查 `results/price_distribution.png` |
| 评分分析图 | 检查 `results/rating_distribution.png` |

**说明：以上是实验记录模板，并非已经执行程序得到的实测数据。** 实际结果会受到网络状况、网站页面结构及请求是否成功等因素影响，应以本地运行结果为准。

### 4. 结果分析

如果程序成功完成采集，CSV 文件中将包含结构化的图书信息，可以直接通过表格软件查看。

价格分布图能够展示采集样本的价格区间和数量分布；评分分布图能够显示不同评分等级对应的图书数量。

需要注意的是，本实验默认只采集前三页，因此分析结果仅代表当前采集到的样本，不能直接代表整个网站的图书分布情况。

### 5. 实验中可能遇到的问题

**问题一：网络请求失败。**

可能与网络连接、DNS 解析、HTTPS 证书验证或网站暂时不可访问有关。可以先检查浏览器是否能够打开测试网站，再根据程序输出的异常信息排查。

**问题二：网页解析不到数据。**

网站页面结构发生变化时，原有 CSS 选择器可能失效。可以使用浏览器开发者工具检查 HTML 结构，并修改相应选择器。

**问题三：CSV 文件中的字符显示异常。**

程序使用 `utf-8-sig` 编码保存文件，一般可以改善 Windows Excel 打开 CSV 时的编码兼容性。

**问题四：没有生成评分图。**

如果所有记录的评分字段都缺失，程序会跳过评分图的生成。应检查网页结构和评分提取逻辑。

## 九、AI 辅助开发过程

本项目使用 AI 工具辅助完成程序设计、代码编写和问题排查。

开发过程中，AI 主要发挥了以下作用：

1. **需求分析：** 将图书信息采集需求拆分为网页请求、数据解析、分页处理、数据清洗和可视化等模块。
2. **代码生成：** 辅助编写 Requests 请求、BeautifulSoup 解析和 Pandas 数据处理代码。
3. **代码优化：** 增加请求超时、异常处理、分页限制和请求间隔，提升程序的稳定性。
4. **代码解释：** 帮助理解 HTTP 请求、CSS 选择器、相对 URL 转换及 CSV 编码等知识。
5. **实验验证：** 根据实际运行输出检查采集数量、字段完整性和文件生成情况。

AI 生成的代码仍需要开发者检查和测试。对于网络爬虫而言，不能只关注代码是否能够运行，还应关注请求频率、网站规则、数据质量和程序异常处理。

## 十、项目总结

通过本次项目，我学习了使用 Python 编写网络爬虫的基本流程，掌握了网页请求、HTML 解析、自动翻页、数据清洗和 CSV 数据导出等技术。

与只提取网页文本的简单爬虫相比，本项目进一步加入了数据统计与可视化功能，使采集到的数据能够用于后续分析。

在开发过程中，AI 辅助编程提高了代码编写和问题排查的效率，但程序最终仍需要通过实际运行来验证正确性。

后续还可以增加多线程请求、SQLite 数据库存储、定时采集、日志记录和 Web 数据展示等功能，将其扩展为更加完整的数据采集系统。

**本项目的核心价值不仅是获取网页数据，更是完成从数据采集、清洗、存储到分析的完整流程。**

---

## 十一、参考资料

1. [Books to Scrape 测试网站](https://books.toscrape.com/)
2. [Requests 官方文档](https://requests.readthedocs.io/)
3. [BeautifulSoup 官方文档](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
4. [Pandas 官方文档](https://pandas.pydata.org/docs/)
5. [Matplotlib 官方文档](https://matplotlib.org/stable/)

## 十二、项目声明

本项目仅用于 Python 网络爬虫学习与课程实验。实际开展数据采集时，应遵守目标网站的使用规则、访问限制及适用法律法规，合理控制请求频率，不绕过身份验证、访问控制或其他安全措施。
