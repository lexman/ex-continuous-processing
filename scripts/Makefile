all: ../data/sp500-companies.csv valid.txt

List_of_S%26P_500_companies.html:
	curl -o List_of_S%26P_500_companies.html "https://en.wikipedia.org/wiki/List_of_S%26P_500_companies"

../data:
	mkdir ../data
    
../data/sp500-companies.csv: ../data List_of_S%26P_500_companies.html scrape-wikipedia.py
	python scrape-wikipedia.py

valid.txt: ../data/sp500-companies.csv ../datapackage.json test_data.py
	python test_data.py
	echo "Datapackage is valid" > valid.txt

.PHONY: all