# Financial SEC Project

# Objective

To get the financial reports of companies in the [S&P 500](https://en.m.wikipedia.org/wiki/List_of_S%26P_500_companies) index from the [SEC](https://en.m.wikipedia.org/wiki/U.S._Securities_and_Exchange_Commission)'s [EDGAR](https://en.m.wikipedia.org/wiki/Electronic_Data_Gathering,_Analysis,_and_Retrieval) portal using web scraping.

# Process

## Getting the list of S&P 500 companies

- Get the list of S&P 500 companies from the [Wikipedia page](https://en.m.wikipedia.org/wiki/List_of_S%26P_500_companies).
- Convert the table from the page to a `.csv` file using the [online tool](https://wikitable2csv.ggor.de/)
  This table contains the CIK values for each company which is very important in the next steps.

## Code

### Setup

```py
from csv import reader
from os import mkdir

from edgarpython.exceptions import InvalidCIK
from edgarpython.secapi import getSubmissionsByCik, getXlsxUrl
from requests import get
from rich.progress import track
```

We are importing the necessary libraries

- `csv` : To work with .csv files in python.
- `os` : To create folders using the `mkdir` function.
- [`edgarpython`](https://github.com/sachin-sankar/edgarpython) : Library to interact with the SEC API.
- `requests` : Library to make HTTP requests.
- `rich` : Library to display progress bars easily.

```py
def download(url, filename):
    resp = get(
        url,
        headers={
            "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:130.0) Gecko/20100101 Firefox/130.0"
        },
    )
    with open(filename, "wb") as file:
        file.write(resp.content)
```

We then define a function `download` which will be used to download the financial reports.
Here we first make use of the `get` method from the requests library to make a HTTP request to the provided file URL then we write the responses content into a file at the given `filename`.

```py
with open("sp500.csv", encoding="utf-8") as file:
    csv = reader(file)
    companies = list(csv)[1:]

mkdir("Output")
```

Using the `reader` class from `csv` library we load the contents of `sp500.csv` file which we created from the Wikipedia page content in to the `companies` variable. We also create a folder called `Output` to store the output of our code.

#### Main Function

```py
for company in track(companies):
    mkdir(f"Output/{company[1]}")
    try:
        submissions = getSubmissionsByCik(company[6])
        selected = []
        for submission in submissions:
            if submission.form == "10-K":
                selected.append(submission)
        print(f"Found {len(selected)} 10-K for {company[1]}")
        downloads = []
        missed = 0
        for submission in selected:
            try:
                downloads.append(getXlsxUrl(company[6], submission.accessionNumber))
            except FileNotFoundError:
                missed += 1
                continue
        print(
            f"{len(downloads)} reports to be downloaded for {company[6]} [missed {missed}]"
        )
        total = len(downloads)
        done = 0
        for downloadUrl in downloads:
            download(
                downloadUrl,
                f"Output/{company[1]}/{downloadUrl.split('/')[-2]}.xlsx",
            )
            done += 1
            print(f"Downloaded [{done}/{total}]")

    except InvalidCIK:
        print("Failed for " + company[1])
        continue
```

1. We go iterate through each company in `companies` list.
2. Create a folder with the name of the company
3. Get all submissions by the company to the EDGAR portal using the `getSubmissionsByCIk` function from the `edgarpython` library.
4. We then filter the submissions and get only the **FORM-10K** reports and put the filtered result into the `selected` list.
5. For each `submission` in the `selected` list we try to see if it has a `.xlsx` file available.
   - If the file is available we add its URL to the `downloads` list.
   - If not we continue to the next submission.
6. For each `downloadUrl` in `downloads` list we download the file using the previously defined `download` function into a folder with the company name we created at step 2.

## Output

The code generates a output of `.xlsx` files separated by each company in their own folder.
The folder structure is as follows.

```
Output
├── 3M
│   ├── 000006674022000010.xlsx
│   ├── 000006674023000014.xlsx
│   ├── 000006674024000016.xlsx
│   ├── 000155837017000479.xlsx
│   ├── 000155837018000535.xlsx
│   ├── 000155837019000470.xlsx
│   ├── 000155837020000581.xlsx
│   └── 000155837021000737.xlsx
├── Abbott Laboratories
│   ├── 000104746918000856.xlsx
│   ├── 000104746919000624.xlsx
│   ├── 000110465920023904.xlsx
│   ├── 000110465921025751.xlsx
│   ├── 000110465922025141.xlsx
│   ├── 000162828023004026.xlsx
│   └── 000162828024005348.xlsx
├── AbbVie
│   ├── 000104746916010239.xlsx
│   ├── 000155115217000004.xlsx
│   ├── 000155115218000014.xlsx
│   ├── 000155115219000008.xlsx
│   ├── 000155115220000007.xlsx
│   ├── 000155115221000008.xlsx
│   ├── 000155115222000007.xlsx
│   ├── 000155115223000011.xlsx
│   └── 000155115224000011.xlsx
├── Accenture
│   ├── 000146737321000229.xlsx
│   ├── 000146737322000295.xlsx
│   └── 000146737323000324.xlsx
```

Sample output printed to the terminal

```
Found 6 10-K for Alexandria Real Estate Equities
6 reports to be downloaded for 0001035443 [missed 0]
Downloaded [1/6]
Downloaded [2/6]
Downloaded [3/6]
Downloaded [4/6]
Downloaded [5/6]
Downloaded [6/6]
```

Each of the `.xlsx` file contain financial statements by each company. Sample from **3M** company's 10K report.

|                                                                                            | Dec. 31, 2021   | Dec. 31, 2020 | Dec. 31, 2019 |
| ------------------------------------------------------------------------------------------ | --------------- | ------------- | ------------- |
| Consolidated Statement of Income - USD ($) shares in Millions, $ in Millions               | 12 Months Ended |               |               |
| Income Statement [Abstract]                                                                |                 |               |               |
| Net sales                                                                                  | $ 35,355        | $ 32,184      | $ 32,136      |
| Operating expenses                                                                         |                 |               |               |
| Cost of sales                                                                              | 18,795          | 16,605        | 17,136        |
| Selling, general and administrative expenses                                               | 7,197           | 6,929         | 7,029         |
| Research, development and related expenses                                                 | 1,994           | 1,878         | 1,911         |
| Gain on sale of businesses                                                                 | 0               | (389)         | (114)         |
| Total operating expenses                                                                   | 27,986          | 25,023        | 25,962        |
| Operating income                                                                           | 7,369           | 7,161         | 6,174         |
| Other expense (income), net                                                                | 165             | 366           | 531           |
| Income before income taxes                                                                 | 7,204           | 6,795         | 5,643         |
| Provision for income taxes                                                                 | 1,285           | 1,337         | 1,114         |
| Income of consolidated group                                                               | 5,919           | 5,458         | 4,529         |
| Income (loss) from unconsolidated subsidiaries, net of taxes                               | 10              | (5)           | 0             |
| Net income including noncontrolling interest                                               | 5,929           | 5,453         | 4,529         |
| Less: Net income (loss) attributable to noncontrolling interest                            | 8               | 4             | 12            |
| Net income attributable to 3M                                                              | $ 5,921         | $ 5,449       | $ 4,517       |
| Weighted average 3M common shares outstanding - basic (in shares)                          | 579             | 577.6         | 577           |
| Earnings per share attributable to 3M common shareholders - basic (in dollars per share)   | $ 10.23         | $ 9.43        | $ 7.83        |
| Weighted average 3M common shares outstanding - diluted (in shares)                        | 585.3           | 582.2         | 585.1         |
| Earnings per share attributable to 3M common shareholders - diluted (in dollars per share) | $ 10.12         | $ 9.36        | $ 7.72        |
