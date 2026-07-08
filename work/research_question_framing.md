# Research Question Framing: Lane 2, Refresh Opportunity Scoring

## Search question
Which pages should be reviewed first for expansion, reduction or refreshing based on changes in visibility and engagement compared to their previous rates.

## Unit of analysis
A weekly observation of each page, identified by its hashed URL.

## Output
A ranked queue containing:
    * Hashed URL: A unique identifier for each page.

    * Opportunity score: A score representing how strongly each page meets criteria that indicate it should be reviewed, used to rank the hashed URLs in the queue.

    * Reason code: The main factors that are contributing to the score such as reduced clicks or low engagement.

    * Suggested action: A recommended action the editor could take during the review such as refresh, expand, prune or monitor the page.

## Potential action
The model could help editors to identify which pages need reviewing most urgently. Editors can then review each page and determine if the suggested action needs to be taken.

## Wrong recommendation cost
    * False positive: The editor potentially wastes time and resources reviewing a page that does not need to be reviewed. This could reduce their trust in the model.

    * False negative: A page with declining performance may go unnoticed, causing its decline in engagement or visibility to continue.

## Why ML can help

Machine Learning can combine multiple factors across different pages such as seasonality; the device being used or the intent behind the searches to create a consistent and scalable method of identifying pages for review.


