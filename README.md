# Source - https://stackoverflow.com/a
# Posted by white
# Retrieved 2026-01-16, License - CC BY-SA 4.0

import urllib.request
import json
import ssl
import os
import time


n = 5  # number of fetched READMEs
url = 'https://api.github.com/search/repositories?q=stars:%3E500&sort=stars'
request = urllib.request.urlopen(url)
page = request.read().decode()
api_json = json.loads(page)

repos = api_json['items'][:n]

for repo in repos:
    full_name = repo['full_name']
    print('fetching readme from', full_name)
    
    # find readme url (case senitive)
    contents_url = repo['url'] + '/contents'
    request = urllib.request.urlopen(contents_url)
    page = request.read().decode()
    contents_json = contents_json = json.loads(page)
    readme_url = [file['download_url'] for file in contents_json if file['name'].lower() == 'readme.md'][0]
    
    # download readme contents
    try:
        context = ssl._create_unverified_context()  # prevent ssl problems
        request = urllib.request.urlopen(readme_url, context=context)
    except urllib.error.HTTPError as error:
        print(error)
        continue  # if the url can't be opened, there's no use to try to download anything
    readme = request.read().decode()
    
    # create folder named after repo's name and save readme.md there
    try:
        os.mkdir(repo['name'])  
    except OSError as error:
        print(error)
    f = open(repo['name'] + '/README.md', 'w', encoding="utf-8")
    f.write(readme)
    print('ok')
    
    # only 10 requests per min for unauthenticated requests
    if n >= 9:  # n + 1 initial request 
        time.sleep(6)
