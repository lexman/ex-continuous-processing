all: valid.txt

List_of_S%26P_500_companies.html:
	curl -o List_of_S%26P_500_companies.html "https://en.wikipedia.org/wiki/List_of_S%26P_500_companies"

../data:
	mkdir ../data
    
../data/sp500-companies.csv: ../data List_of_S%26P_500_companies.html scrape-wikipedia.py
	python scrape-wikipedia.py

valid.txt: ../data/sp500-companies.csv ../datapackage.json test_data.py
	python test_data.py
	echo "Datapackage is valid" > valid.txt

pushed.txt: valid.txt
	git add ../data/sp500-companies.csv
	git commit -m "[data][skip ci] automatic update" || exit 0
	git push publish
	echo "Update has been pushed if there was a change" > pushed.txt

.PHONY: all