+++
date = "26 Mar 2014"
title = "Export all Google Sheets to csv"
draft = false
url = "/posts/export-all-google-sheets-to-csv"
tags = ["google sheets", "csv"]
image = ""
comments = true
share = true
menu= ""
author = "Michael DeRazon"
aliases = [
    "/export-all-google-sheets-to-csv"
]
+++

I had a Google Apps spreadsheet with around 40 sheets. Wanted to export them to `csv`. Unfortunately Google only lets you export one sheet at a time.

Here's a script that exports all sheets in a spreadsheet to `csv`

<script src="https://gist.github.com/mderazon/9655893.js"></script>



To use, click on tools --> script editor. Then create a new blank script and paste the code there.

Click on the play button (run) and go back to your spreadsheet. You should then see the csv in the spreadsheet menu
