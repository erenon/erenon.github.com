---
title: "LedgerSplit vim function"
date: 2023-03-09
---

I manage my finances using [ledger][].
Periodically, I export my bank statements and push them through a [make][]
pipeline to get a ledger file. The conversion assigns transactions
to accounts based on a set of rules, e.g: If the payee is "Foo Shop",
the account is going to be "Expenses:Food":

```ledger
2023-03-09 Foo Shop
    Expenses:Food           1000 HUF
    Assets:Card
```

However, sometimes I buy e.g: toothpaste in "Foo Shop", that I don't want to eat.
I do not want to contaminate my ledger, so I save the receipt and mark the item.
During the periodic export, I manually fix the ledger file in [vim][]:

```ledger
2023-03-09 Foo Shop
    Expenses:Food            750 HUF
    Expenses:Health          250 HUF
    Assets:Card
```

Since I have an engineering degree, most of the time I get this subtraction right.
For the same reason, I made a tiny vim script, that automates some of this.
Standing on this line:

```ledger
    Expenses:Food           1000 HUF
```

Issuing this command:

```
:LedgerSplit 100+200
```

Modifies the posting like this:

```ledger
    Expenses:Food           700 HUF
    Expenses:Food           300 HUF
```

It splits the given amount from the current posting, and moves it to the next line.
It handles expressions (as shown) and decimal fractions.
After this I just need to fix the second account name.

## Installation

To `.vimrc`, add:

```vim
au BufRead,BufNewFile *.ledger set filetype=ledger
```

To `.vim/after/ftplugin/ledger.vim`:

```vim
function! LedgerSplit(amount) abort
  let pat = "[0-9\\.]\\+"
  let line = getline(".")
  let old = str2float(matchstr(line, pat, 0))
  let new = substitute(printf("%.2f", old-a:amount), "\\.00$", "", "")
  let split = substitute(printf("%.2f", a:amount), "\\.00$", "", "")
  call setline(".", substitute(line, pat, new, ""))
  call append(".", substitute(line, pat, split, ""))
endfunction

command! -nargs=1 LedgerSplit call LedgerSplit(<args>)
```


[ledger]: https://www.ledger-cli.org/
[make]: https://www.gnu.org/software/make/
[vim]: https://www.vim.org/
