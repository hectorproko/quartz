---
tags:
  - "#Python"
title: XSOAR Python Transformer Script
---




`consolidate_null_items`
```python
js = demisto.args()["value"] #this is string
js = json.loads(js) #turning above to dict

null_items = "" #to hold list as a string
for key, value in js.copy().items():
  if js[key] == "null":
    #print(value, key)
    if null_items == "":
      null_items = key
    else:
      null_items += "," + key
    js.pop(key)

js["Null Items"] = null_items #Adding to the dictionary

demisto.results(js)
```


### Script Explanation:

This code defines a variable called `js` which is assigned the value of the `value` argument passed to the function. This value is expected to be a string containing a JSON object.

Next, the code uses the `json.loads()` function to convert the string into a Python dictionary. This allows us to access and manipulate the data contained in the JSON object as a dictionary in Python.

The code then defines a variable called `null_items` which is initialized to an empty string. This string will be used to hold a list of keys from the dictionary whose values are equal to the string "null".

The code then uses a `for` loop to iterate over the items in the dictionary. For each item, it checks whether the value is equal to "null". If it is, it adds the key to the `null_items` string.

> [!tip]
> We can append keys to the empty string from the beginning without needing the initial check (`if null_items == ""`). The reason the original code includes this check is to format the `null_items` string in a specific way - to ensure there are no leading commas before the first key.

After all items in the dictionary have been processed, the code adds a new item to the dictionary with the key "Null Items" and the value of the `null_items` string. Finally, the code uses the `demisto.results()` function to return the modified dictionary as the result of the function.
