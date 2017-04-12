+++
date = "31 Dec 2013"
title = "Export mongodb collections to csv without specifying fields"
draft = false
url = "/posts/export-mongodb-collections-to-csv-without-specifying-fields"
tags = ["mongodb", "mongoexport", "csv"]
image = ""
comments = true
share = true
menu= ""
author = "Michael DeRazon"
aliases = [
    "/export-mongodb-collections-to-csv-without-specifying-fields"
]
+++

Sometimes you might want to export your Mongodb database in `csv` format. If you have many collections, it can be a bit of a hassle since [mongoexport](http://docs.mongodb.org/v2.2/reference/mongoexport/) requires you to specify each field you want to be exported with the `--fields` attribute and you have to do it for each collection seperately.

Here's a little bash script that will do the work for you.

<script src="https://gist.github.com/mderazon/8201991.js"></script>

Result is a `.csv` file for each collection you have in your db.
