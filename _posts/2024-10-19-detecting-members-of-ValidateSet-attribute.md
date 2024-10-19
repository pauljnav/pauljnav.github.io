# Programmatically detecting the members of a ValidateSet attribute within your function.

Well, I struggled with this one, so best I preserve this story into the web.

I had set out to develope a reporting module/function and I needed to define a function containing a large list of ValidateSet() options.
_I measured 17 of them_

The way i wanted the function to work was to allow the user to select one or many valid options. Provide for tab completion. ANd to define the function in a way that easily selected all 17 options because that is the default behaviour of the reporting application that I was developing for.

I had begun to define the function by using an extra '*' as an option to implement `all` valid options, but I caught myself in time  and the better design is to let that be the default behaviour for that parameter when the user makes no parameter selection at all.

In this way,the user can select option1, or option1 with option2 etc, but when they select no option, they get all 17 valid options by default. Afer all, the default behaviour for the reporting I am developing for to have all 17 options selected, but the user needs the capability to select only what they need.
