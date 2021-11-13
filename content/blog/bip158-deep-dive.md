---

title: "Compact Block Filters Deep Dive (BIP 158)"
subtitle: "A technical explanation of the workings of compact block filters and
Golomb-Rice Coding."
date: 2021-11-13T09:32:00+02:00
authors:
  - name: Elle Mouton
    github: ellemouton
    twitter: ellemouton
tags:
  - "BIP158"
  - "Compact block filters"
  - "Bitcoin light client"

images:
  - "/post-data/compact-block-filters/cover.png"

appearedfirston:
    label: "ellemouton.com"
    url: "https://ellemouton.com/posts/bip158/"

---

In this post, I will briefly describe the needs of a bitcoin light client and
why compact block filters satisfy these needs better than Bloom filters do. Then
I will dive into exactly how compact block filters work and will follow this
with a step-by-step guide for constructing such a filter from a testnet block.

#### The purpose of block filters

A bitcoin light client is software that can back a bitcoin wallet without
storing the blockchain. This means that it needs to be able to broadcast
transactions to the network, but most importantly, it must be able to pick up
when there is a new transaction that is relevant to the wallet it is backing.
There are two ways a transaction becomes relevant to a wallet: either it is
sending money to the wallet (creating a new output to a wallet address), or it
is spending one of the UTXOs that the wallet owns.

#### What was wrong with Bloom filters?

Before [BIP 158] came along, the most widely used method for light clients was
to use Bloom filters[^bloomfilters] as described in [BIP 37]. With a bloom
filter, you take all the objects you are interested in (script pub keys spent or
created), hash them a couple times and add the result of each to a bit map
called a Bloom filter. This filter represents what you are interested in. You
would then send this filter to a trusted bitcoin node and ask them to send you
anything that matches your filter. The problem with this is that it is not very
private since you are revealing some information to the bitcoin node you are
sending this filter to. They can start getting an idea of the transactions you
are interested in as well as the ones you are definitely not interested in. They
can also just decide not to send you a transaction that matches the filter. So
as you can see, it isn’t great for the light client. But it is also not great
for the bitcoin node serving the light client. Each time you send them a filter,
they have to load the relevant block from disk and determine which transactions
match your filter. You could just spam them with fake filters and effectively
DOS them. It takes very little energy to create a filter and lots to respond to
it.

#### Introducing Compact Block Filters:

Ok, take two. What we want is:
- More privacy
- Less asymmetry in the client - server work load. Ie, the server should be
  required to do way less work. 
- Less trust. The light client shouldn't need to worry about the server holding
  back relevant transactions.

With compact block filters, the server (full node) will for each block construct
a deterministic filter that includes all the objects in the block. This filter
can be calculated once and persisted. If light clients request a filter for a
block, there is no asymmetry since the server won't have to do any more work
than the client had to do when making the request.  A light client can also
choose to download the filters from multiple sources to ensure they match and
can always download the full block and check for itself if the filter that the
server provided was indeed correct given the block's contents. Another bonus is
that this is way more private. The light client no longer sends a fingerprint of
the data it is interested in to the server. And so it becomes way more difficult
to analyse the light client's activity. The light client gets these filters from
the server and checks for itself if any of its objects match what is seen in the
filter, and if it does match, then the light client asks for the full block. One
thing to note with this way of doing things is that full nodes serving the light
clients will need to persist these filters, and the light clients might also want
to persist a few filters and so it is important that the filters are as small as
possible (hence the name, compact block filters).

Cool! Now we get to the cool stuff. How is this filter created? What does it
look like? 

What do we want? 
- We want to put fingerprints of certain objects in the filter so that when
  clients are looking to see if a block maybe contains info relevant to them,
they can take all their objects and check if the filter matches on those
objects.
- We want the filters to be as small as possible.
- Effectively we want to sort of summarise some of the block info… in a size
  much much smaller than the block.

The info included in the basic filter is: every transaction's input's
scriptPubKey being spent and every transaction's output's scriptPubKey being
created. So something like this:
```go
objects = {spk1, spk2, spk3, spk4, ..., spkN} // A list of N scriptPubKeys.
```

Technically we could just stop here and say this list of scriptPubKeys is our
filter. It is a condensed version of what is in the blockchain and contains the
info the light client needs. With this list, they could tell with 100% certainty
if something they are interested in is in the block. But it is still pretty big.
So the next step is all about making this list as compact as possible. This is
where things get insanely cool.

First, we convert each object into a number in a range such that the object
numbers are uniformly distributed in that range: Let's say we have 10 objects (N
= 10), then we have some function that turns each of the objects into a number.
Let’s say we chose the range [0, 10] since we have 10 objects. Now the
hashing-plus-convert-to-number function we use will take each object and produce
a number in the space from [0, 10]. It is uniformly distributed in this space.
That means that, after ordering them, we will get (in the very, very ideal case)
something like this:

{{< center-figure src="/post-data/compact-block-filters/dense.jpeg" caption=""
height=250 width=250 >}}

First of all, wow that is so great cause we have drastically decreased the size
of an objects fingerprint. Each one is just a number now. Ok so, let this be
our new filter:

```go
numbers := {1,2,3,4,5,6,7,8,9,10}
```

Now a light client downloads the filter and wants to see if one of the objects
they are looking for is matched in this filter. All they need to do is take
their objects and do the same hashing-plus-convert-to-number scheme and check if
any of the numbers are in the filter. What is the problem? The filter has a
number for each possible number in the space! Meaning that absolutely any object
will match on this filter. In other words, the false-positive rate of this
filter is 1. This is no good. We have lost too much info on our quest to
compress the data in the filter. What we need is a higher false-positive (fp)
rate. Ok so let’s say we want a false positive rate of 5. Then what we want is
to have our objects be mapped uniformly to a space of [0, 50]:

{{< center-figure src="/post-data/compact-block-filters/sparse.png" caption=""
height=250 width=250 >}}

This is starting to look a bit better. If I am a client downloading this filter
and I check if my objects are maybe in the filter, there will be a 1/5 chance
that if it matches, it is a false positive. Great, so now we have mapped 10
objects to numbers between 0 & 50. This new list of numbers is our filter.
Again, we could stop here… but we can compress this even further!!

We have this list of ordered numbers that we know are distributed uniformly
across this space between [0, 50]. We know that there are 10 items in the list.
What this means is that we can deduce that the most likely _difference_ between
each of the numbers in this ordered list is about 5. In general, if we have N
items and a false positive rate of M, then the space will be of size N * M. So
the numbers in the space can range from 0 to N * M, but the difference between
each number (once ordered) will be roughly M. M will definitely be a smaller
number to store than a number in the N * M space. So what we can do is instead
of storing each number, we can instead store the difference of each successive
number. In the above case, this would mean that instead of storing `[0, 5, 10,
15, 20, 25, 30, 35, 40, 45, 50]`, we just store `[0, 5, 5, 5, 5, 5, 5, 5, 5, 5]`
and then it is trivial to reconstruct the original list. As you can gather,
storing the number 50 requires way more bits than storing the number 5. But why
stop there? We can compress this even further!

This is where Golomb-Rice Coding comes in. This encoding works well for a list
of numbers that will all very likely be close to some number. This is what we
have! We have a list of numbers that will all very likely be close to 5 (or, in
general, close to our FP rate of M) and so taking the quotient of any number in
the list with that number (dividing each number by 5 and ignoring the remainder)
will very likely be 0 (if the number is slightly less than 5) or 1 if the number
is slightly more than 5. The quotient could be 2, 3, etc, but the likelihood
decreases a lot. Great! So we can take advantage of this knowledge and say that
we will encode a small quotient with the smallest number of bits that we can and
use more bits to encode larger, unlikely quotients. Then we also need to encode
the remainders (since we want to be able to reconstruct the values exactly), and
these will always be numbers between [0, M-1] (in our case, [0, 4]). For
encoding the quotients, we use the following mapping:

{{< center-figure src="/post-data/compact-block-filters/quotient.png" caption="" height=125 width=125 >}}

The mapping above is easy to read: The number of `1`s indicates the quotient we
are encoding, and the `0` indicates the end of the quotient encoding. So for each
number in our list, we encode the quotient using the above table, and then we
convert the remainder to binary using the number of bits needed to encode the
maximum of M-1. In our case, that is 3 bits. Here is a table showing the
encoding of the possible remainders in our example:

{{< center-figure src="/post-data/compact-block-filters/remainder.png" caption="" height=125 width=125 >}}

So, in our ideal case example, our list of `[0, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5]`
can be encoded as follows:
```go
0000 10000 10000 10000 10000 10000 10000 10000 10000 10000
```

Before we move on to a more realistic example, let’s see if we can reconstruct
our original list from this filter.

Ok so we have: “0000100001000010000100001000010000100001000010000”. We know how
Golomb-Rice Coding encodes quotients and we also know that M is 5 (since this
will be public knowledge known to everyone using this filter construction).
Since we know M is 5, we know that 3 bits will be used to encode the remainders.
So we can take our filter and produce the following quotient-remainder tuples:

```go
[(0, 0), (1, 0), (1, 0), (1, 0), (1, 0), (1, 0), (1, 0), (1, 0), (1, 0), (1, 0)]
```

We know that the quotients were produced by dividing the numbers by M (5), so we can
reconstruct these:

```go
[0, 5, 5, 5, 5, 5, 5, 5, 5, 5]
```

And we know that this list represents differences of numbers, so we can
reconstruct the OG list:

```go
[0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50]
```

#### A more realistic example

We will now try to construct a filter from an actual Bitcoin testnet block. I'm
going to use block [2101914]. Let’s see what its actual filter is:

```
$ bitcoin-cli getblockhash 2101914
000000000000002c06f9afaf2b2b066d4f814ff60cfbc4df55840975a00e035c

$ bitcoin-cli getblockfilter 000000000000002c06f9afaf2b2b066d4f814ff60cfbc4df55840975a00e035c
```
```json
{
  "filter": "5571d126b85aa79c9de56d55995aa292de0484b830680a735793a8c2260113148421279906f800c3b8c94ff37681fb1fd230482518c52df57437864023833f2f801639692646ddcd7976ae4f2e2a1ef58c79b3aed6a705415255e362581692831374a5e5e70d5501cdc0a52095206a15cd2eb98ac980c22466e6945a65a5b0b0c5b32aa1e0cda2545da2c4345e049b614fcad80b9dc9c903788163822f4361bbb8755b79c276b1cf7952148de1e5ee0a92f6d70c4f522aa6877558f62b34b56ade12fa2e61023abf3e570937bf379722bc1b0dc06ffa1c5835bb651b9346a270",
  "header": "8d0cd8353342930209ac7027be150e679bbc7c65cc62bb8392968e43a6ea3bfe"
}
```

Ok shweeeeeet, let’s see if we can reconstruct this filter from the block.

The full code for this can be found in this [github repo]. I will just show some
pseudo code snippets here. The beef of the code is the function called
`constructFilter`, which takes in a bitcoin client that can be used to make calls
to bitcoind and the block in question. The function looks something like this:


```go
func constructFilter(bc *bitcoind.Bitcoind, block bitcoind.Block) ([]byte, error) {
	// 1. Collect all the objects from the block that we want to add to the filter

	// 2. Convert all the objects to numbers and sort them. 

	// 3. Get the differences between the sorted numbers

	// 4. Encode these differences using Golomb-Rice Coding
}
```

Ok so step 1 is to collect all the objects from the block that we want to add to
the filter. From the BIP, we know that these objects are all the scriptPubKeys
being spent as well as all the scriptPubKeys of each output. Some extra rules
from the BIP are that we skip the input for the coinbase transaction (since it
is empty and meaningless), and we skip any OP_RETURN outputs. We also
de-duplicate the data. So if there are two identical scriptPubKeys, we only
include one in the filter.


```go
// The list of objects we want to include in our filter. These will be
// every scriptPubKey being spent as well as each output's scriptPubKey.
// We use a map so that we can dedup any duplicate scriptPubKeys.
objects := make(map[string] struct{})

// Loop over every transaction in the block.
for i, tx := range block.Tx {

        // Add the scriptPubKey of each of the transaction's outputs
        // and add those to our list of objects.
        for _, txOut := range tx.Vout {
                scriptPubKey := txOut.ScriptPubKey

                if len(scriptPubKey) == 0 {
                        continue
                }

                // We don't add the output if it is an OP_RETURN (0x6a).
                if spk[0] == 0x6a {
                        continue
                }

                objects[skpStr] = struct{}{}
        }

        // We don't add the inputs of the coinbase transaction.
        if i == 0 {
                continue
        }

        // For each input, go and fetch the scriptPubKey that it is
        // spending.
        for _, txIn := range tx.Vin {
                prevTx, err := bc.GetRawTransaction(txIn.Txid)
                if err != nil {
                        return nil, err
                }

                scriptPubKey := prevTx.Vout[txIn.Vout].ScriptPubKey

                if len(scriptPubKey) == 0 {
                        continue
                }

                objects[spkStr] = struct{}{}
        }
}

```

Ok great, we have all the objects we care about. And now we can also define the
variable N to be the length of the `objects` map. In this example, N is 85.

The next step is to convert each of the objects to numbers spread uniformly
across a range. Remember that this range depends on the false-positive rate we
want. BIP158 defines the constant M to be 784931. This means that we want every
1/784931 matches to be a false-positive. As we did in our earlier example, we
take this fp rate of M and multiply it by N to get the range that we want all
our numbers to lie in. We define this as F where F = M\*N. In our case, we have
85 objects, and so F=66719135. I am not going to go into the details of the
function used to map our objects to numbers (you can check out the details of
this in the code in the linked repo). All you need to know for now is that it
takes in an object, the constant F, which defines the range that it needs to map
the object to, and a key which is the block hash. Once we have all the numbers,
we sort the list in ascending order, and then we also create a new list called
`differences` which will hold the differences between each sequential number in
the sorted `numbers` list.

```go
numbers := make([]uint64, 0, N)

// Iterate over all the objects, convert them to numbers lying uniformly in the 
// range [0, F] and add them to the `numbers` list.
for o := range objects {
        // Using the given key, max number (F) and object bytes (o),
        // convert the object to a number between 0 and F.
        v := convertToNumber(b, F, key)

        numbers = append(numbers, v)
}

// Sort the numbers.
sort.Slice(numbers, func(i, j int) bool { return numbers[i] < numbers[j] })

// Convert the list of numbers to a list of differences.
differences := make([]uint64, N)
for i, num := range numbers {
        if i == 0 {
                differences[i] = num
                continue
        }

        differences[i] = num - numbers[i-1]
}
```

Awesome! Here is a graph showing the values in the `numbers` and `differences`
lists:

{{< center-figure src="/post-data/compact-block-filters/bitcoinExample.png" caption="" height=600 width=600 >}}

As you can see, the 85 numbers are really nicely uniformly distributed across
the space! And this results in the values in the `differences` list being pretty
small. 

The last step now is to use Golomb-Rice Coding to encode this `differences`
list. Recall from the earlier explanation that we need to divide each difference
by its most likely value and then we encode that quotient along with the
remainder. In my earlier example, I said that this most-likely value would be the
M that we choose and that the remainder would then lie in the range [0, M].
However, this is not what is done in the BIP as it was found[^golombloss] that
this is, in fact, not the ideal way to choose the Golomb-Rice coder parameter when
trying to optimize for the smallest possible size of the final encoded filter.
And so, instead of using M, a new constant of P is defined and P^2 is used as the
Golomb-Rice parameter. P is defined as 19. This means that each difference value
is divided by 2^19 to get the quotient and remainder and the remainder is then
encoded in binary in 19 bits.

```go
filter := bstream.NewBStreamWriter(0)

// For each number in the differences list, calculate the quotient and
// remainder after dividing by 2^P.
for _, d := range differences {
        q := math.Floor(float64(d)/math.Exp2(float64(P)))
        r := d - uint64(math.Exp2(float64(P))*q)

        // Encode the quotient.
        for i := 0; i < int(q); i++ {
               filter.WriteBit(true)
        }
        filter.WriteBit(false)

        filter.WriteBits(r, P)
}
```

Great stuff! Now when we print out this filter, we get: 

```
71d126b85aa79c9de56d55995aa292de0484b830680a735793a8c2260113148421279906f800c3b8c94ff37681fb1fd230482518c52df57437864023833f2f801639692646ddcd7976ae4f2e2a1ef58c79b3aed6a705415255e362581692831374a5e5e70d5501cdc0a52095206a15cd2eb98ac980c22466e6945a65a5b0b0c5b32aa1e0cda2545da2c4345e049b614fcad80b9dc9c903788163822f4361bbb8755b79c276b1cf7952148de1e5ee0a92f6d70c4f522aa6877558f62b34b56ade12fa2e61023abf3e570937bf379722bc1b0dc06ffa1c5835bb651b9346a270
```

Apart from the first two bytes, this matches the filter we got from bitcoind
exactly! Why the 2 byte difference? The BIP says that the N value needs to be
encoded in CompactSize format and appended to the front of the filter so that it
can be decoded by the receiver. This is done as follows:

```go
fd := filter.Bytes()

var buffer bytes.Buffer
buffer.Grow(wire.VarIntSerializeSize(uint64(N)) + len(fd))

err = wire.WriteVarInt(&buffer, 0, uint64(N))
if err != nil {
        return nil, err
}

_, err = buffer.Write(fd)
if err != nil {
        return nil, err
}
```

If we print out the filter now, we get one that matches the one we got from
bitcoind exactly:

```
5571d126b85aa79c9de56d55995aa292de0484b830680a735793a8c2260113148421279906f800c3b8c94ff37681fb1fd230482518c52df57437864023833f2f801639692646ddcd7976ae4f2e2a1ef58c79b3aed6a705415255e362581692831374a5e5e70d5501cdc0a52095206a15cd2eb98ac980c22466e6945a65a5b0b0c5b32aa1e0cda2545da2c4345e049b614fcad80b9dc9c903788163822f4361bbb8755b79c276b1cf7952148de1e5ee0a92f6d70c4f522aa6877558f62b34b56ade12fa2e61023abf3e570937bf379722bc1b0dc06ffa1c5835bb651b9346a270
```

Yay!

However, from my understanding, there is no need to add N to the filter. If you
know the value of P, then you can figure out the value of N. Let’s do this now
by seeing if we can take the filter above, and reconstruct the original list of
numbers:

```go
b := bstream.NewBStreamReader(filter)
var (
	numbers []uint64
	prevNum uint64
)

for {

        // Read a quotient from the stream. Read until we encounter
        // a '0' bit indicating the end of the quotient. The number of 
        // '1's we encounter before reaching the '0' defines the 
        // quotient.
        var q uint64
        c, err := b.ReadBit()
        if err != nil {
                return err
        }

        for c {
                q++
                c, err = b.ReadBit()
                if errors.Is(err, io.EOF) {
                        break
                } else if err != nil {
                        return err
                }
        }

        // The following P bits are the remainder encoded as binary.
        r, err := b.ReadBits(P)
        if errors.Is(err, io.EOF) {
                break
        } else if err != nil {
                return err
        }

        n := q*uint64(math.Exp2(float64(P))) + r

        num :=  n + prevNum
        numbers = append(numbers, num)
        prevNum = num
}

fmt.Println(numbers)
```

The above produces the same list of numbers that we had before and we were able
to reconstruct this without the knowledge of N. So I am not sure why it was
decided that N should be added to the filter. If anyone knows why it was
required to add N to the filter, please let me know!

Cool, that was fun! Thanks for reading. This is a [cross-post] from my
[website], where you can find many more Bitcoin and Lightning related technical
posts.  Until next time... Yeeeeet!

[github repo]: https://github.com/ellemouton/bip158Example
[cross-post]: https://www.ellemouton.com/blog/view/9
[website]: https://www.ellemouton.com
[BIP 158]: https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki 
[BIP 37]: https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki 
[2101914]: https://blockstream.info/testnet/block/000000000000002c06f9afaf2b2b066d4f814ff60cfbc4df55840975a00e035c


[^bloomfilters]: https://en.wikipedia.org/wiki/Bloom_filter
[^bip37]: https://github.com/bitcoin/bips/blob/master/bip-0037.mediawiki
[^repo]: https://github.com/ellemouton/bip158Example
[^golombloss]: https://gist.github.com/sipa/576d5f09c3b86c3b1b75598d799fc845


