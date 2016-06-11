all: pushed.txt

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
	git commit -m "[data] automatic update" || exit 0
	git pull --rebase
	git push
	echo "Update has been pushed (if there was some change)" > pushed.txt

.PHONY: all