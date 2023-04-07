## The database

The database is an essential part of the node. As you may know, the majority of cryptocurrencies uses a Blockchain, a time segregating, one dimensonial, acyclic directed graph. Preservating, sharing and updating this structure is mandatory for any cryptocurrency to work.
While we could idiomatically implement a simple folder with a file for each block, the reality is that such an approach would be extremely slow. A node absolutely needs to quicky gather the data it need to operate its protocol. And the best approach towards quickly manipulating data structures, that are dynamic by nature, is a database.

How we implement the database ? what storage engine to use ? what are the underlying separation between data ?
These are all important questions that play a role in the efficiency or performance of a node.

The ongoing chapters explain monerod and cuprate's database design. How it diverges and why we made our choices.

The ongoing chapter can use general concept of Monero, that can't be shortly re-explain. In this case, we'll just leave a link to the corresponding documentation in Monero's part of the book.