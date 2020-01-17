Say that you've connected to a bunch of remote peers and emitted a log message for each
incoming message:

```bash
$ cat msgs
message=GetBlockHeaders remote=0x029526@202.79.167.70 block_number_or_hash=9043967
message=Transactions remote=0x029526@202.79.167.70 count=472
message=NewBlockHashes remote=0x029526@202.79.167.70 blocknum=9284688
message=NewBlockHashes remote=0x029526@202.79.167.70 blocknum=9284689
message=NewBlockHashes remote=0x029526@202.79.167.70 blocknum=9284690
message=NewBlockHashes remote=0x029526@202.79.167.70 blocknum=9284691
message=Transactions remote=0x1b9439@80.66.80.22 count=8
message=NewBlockHashes remote=0x1b9439@80.66.80.221 blocknum=9593232
```

Now I want to ask, how many transactions did I receive in total? That's an easy question
for any query language I know of:

```bash
$ awk '/Transactions/{split($3, arr, "="); sum += arr[1]} END {print sum}' msgs
```

But there's a catch! You've accidentally connected to too many nodes. The ones which are
advertising blocks in the 9500000 range actually belong to a different network, we don't
care about them. What we really want to know is how many transactions did I receive in
total, _from nodes which also advertised a good block_. This is harder!

Expressing this in awk requires getting bogged down in the mechanics. You cann build an
associative array (dict) totaling transactions received from each remote, as well as an
array marking the good peers. Then, on END, you can sum up the subtotals.

SQL has an easier time (especially if we ignore creating and loading data into tables),
the solution reads almost exactly like the problem description:

```sql
SELECT count(txn_count) FROM messages WHERE message = 'Transaction' AND remote IN (
  SELECT remote FROM messages WHERE message = 'NewBlockHashes' AND blocknum < 9500000
);
```

(In lieu of breaking out something like JSON syntax, imagine these rows have all been
 thrown into a table with a lot of columns which are empty if they don't apply to the row's
 message type. proper database design is off topic here).

Actually, shell also has a more relational answer:

```bash
$ join <(awk '/Transactions/{print $2, $1, $3}' msgs) \
       <(grep num=92 msgs | awk '{print $2}' | uniq)  \
    | awk '{split($3, arr, "="); sum += arr[2]} END {print sum}'
472
```

It takes a bit of futzing around, someone who knows SQL could read the SQL query far
faster than someone who knows shell could read this one. It's also at a bit of a
disadvantage, awk expects values to be alone in their columns, the key=value syntax is
foreign. And finally, let's not yet talk about how quickly each of these queries run.

However, it's cool that shell supports the relational style at all!

Just for fun, what would the equivalent datalog query look like?

TODO: it's too easy to make a hypothetical language look clean, for fairness I should
      probably write a datomic query here.

```prolog
> good_node(Remote) :- message(Msg), member(message=new_block_hash, Msg),
                       member(blocknum=Num, Msg), Num < 950000,
                       member(remote=Remote, Msg).
> findall(
    Count,
    (
      message(Msg), member(message=transaction, Msg),
      member(remote=Remote, Msg), good_node(Remote),
      member(count=Count, Msg),
    ),
    Counts
  ), sum_list(Counts, Total).
```

Well, I had to fall back to writing Prolog. Datalog doesn't support aggregation and also
doesn't support records (represented here as a list of key=value pairs), but I think
there's something here worth exploring, with just a little work this could be just as
natural as the SQL query was.

You could imagine a language/configuration where it worked like this;

```
where
  good_node(Remote): new_block_hash(Remote, Blocknum), Blocknum<9500000.
  count(Count): transaction(Remote, Count), good_node(Remote).
find sum(count(Count)).
```
