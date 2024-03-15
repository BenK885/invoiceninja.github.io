---
extends: _layouts.user_guide 
section: content
locale: en
---

# Hosted TroubleShooting

## Auth.net

### CardCode is required.. (or similar)

If you are seeing errors when processing payments with a return error indicating a missing field such as

<x-info>
There was an error processing your payment [33]. Please try again later.  
Message:  
Customer ID is required  
</x-info>

This means that you have a payment form configured within Auth.net with some required fields. These will need to be disabled.

To do this, log into auth.net and navigate to 

```
Account / Payment Forms / Form Fields
```

You will see a list of fields with check boxes like this:

![alt text](/assets/images/troubleshooting/authnet-required-fields.png "Required Fields")

<x-warning>
Disable the required checkboxes and then save.
</x-warning>