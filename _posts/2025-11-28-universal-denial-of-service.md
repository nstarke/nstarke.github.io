---
layout: posts
title: "Universal Denial of Service"
date: 2025-11-28 00:00:00 -0600
categories: dos denial-of-service weird
author: Nicholas Starke
---

## Forward

I have not written much on this blog this year as I have been pursuing other endeavors. 

I have been toying around with a few ideas over the last few weeks, and I thought I would try my hand at writing a different type of blog post. My intent with this post is to glue together a few different ideas from math, physics, and computer security to produce an interesting take that is mostly meant for entertainment. It is as much science fiction as anything else. 

If this post produces any sort of conversation or just gets people thinking, it will have achieved its goal. 

## Universal Denial of Service

### Computable Matter
Computronium [1] is a general term used to describe highly efficient computable matter. Now this does not exist at present time in any capacity, so we are starting at a point of pure speculation. The idea is to use atoms or sub-atomic particles as computation “devices”.  These devices can theoretically calculate anything a classical computer can compute, and there have been extensions of the idea proposed to handle quantum computing as well. 

Let’s say we use some advanced technology (maybe nanotechnology) to convert a gram of matter into computronium. We can then use the computronium to perform computable calculations. We can write programs that can execute on the gram of computronium and return to us a result. 

But what if the program we write requires more compute resources than are available in the gram of matter? It is not a huge leap to say that if we can create a gram of Computronium, we can create another gram. This might even be an automated process carried out by the Computronium itself. For the sake of this idea, we are going to assume our gram of Computronium has the property of being able to produce arbitrary amounts of new Computronium from surrounding ordinary matter to meet the resource needs for the currently running program. 

### Denial of Service

In computer security, a denial of service (DoS) condition occurs when legitimate users cannot access resources they need to fulfill a task due to the actions of another user. When these conditions are created purposefully, they are considered a Denial of Service attack. Usually these attacks are some form of resource exhaustion; whether it be network bandwidth as is common with botnets [2] in a Distributed Denial of Service (DDoS) attack. In these types of attacks, large amounts of hijacked, network-connected computers target one specific network-connected computer or sub network and send as much unique traffic as they can to that location. The result is that the target becomes inaccessible for legitimate users of the target system. 

A denial of service vulnerability is some type of computer hardware or software flaw that can be exploited to deny legitimate users access to needed resources. This can take many forms; severe examples are memory corruption vulnerabilities that do not result in arbitrary code execution , but do result in a process crashing. This is particularly severe when the vulnerability can be exploited over the network. Double that severity if there is no way for the operating system to restart the process from a good state. Double it again if the memory corruption vulnerability occurs in the operating system itself [2] resulting in the operating system panicking (I have personally discovered remotely exploitable kernel memory corruption vulnerabilities that resulted in the target host OS panicking and not being able to even reboot, so I know they do exist even though they are rare). 

I would argue the class of most severe denial of service vulnerability is that of the “halt and catch fire” bug. For example, many years ago there was a processor opcode (f00f) in certain Intel processors [3] that when executed resulted in the CPU locking up and failing to execute any additional instructions until the power was reset on the computer [4]. When the CPU attempted to execute the faulty CPU instruction, the computer entered into an undefined state which it could not recover from without physical intervention by a user. 

These sorts of attacks are generally done purposefully by a threat actor, but one could argue denial of service conditions can be triggered by accident. Take any of the major cloud service provider outages [5] [6] [7] [8] in the last few years; legitimate users were not able to access network and computer resources not because of the malicious actions of some threat actor, but because of some internal cascading error that took down a cloud provider data center somewhere. 

How do denial of service attacks relate to Computronium? Can a computer made of matter operating at the atomic or subatomic level be used to create a denial of service against legitimate users? What would that even look like? How could such a condition occur? If it can occur at all, could a threat actor purposefully exploit that condition?

### Big Numbers

Let’s take a break from the topic of computer security to talk about numbers; specifically large numbers. Most people have heard of the term googol [9], and probably even googolplex [10]; nerds like me may even be familiar with numbers like Graham’s number [11], which is a number that requires special notation created specifically for this number to express because it is physically impossible to express using any pre-existing conventional notation. Graham’s number is so enormous it cannot be expressed in conventional form because even if we use atoms to write out the number, there are not enough atoms in the universe to write it out. Not even close. 

However, since the discovery of Graham’s number in 1971 [12], mathematicians and logicians have found even larger numbers.  For example, one is Rayos’ Number [13] (denoted “R(100)”). These numbers involve self-referential and meta-modifying definitions - and those do not translate well into computational tasks. As such, it would be hard for anyone to write a computer program to calculate Rayos’ number. 

Enter TREE(3) [14]. TREE(3) is based on a paper from 1960, and a good summary can be found here [15]. TREE(3) is a solution to a problem in graph theory that results in an enormous number, and it’s basically the largest number used in a serious mathematical sense that has been discovered to date by people. It has a few very interesting properties as it relates to computers. 

First, the problem that TREE(3) is a result of is simple to describe using a set of instructions; there are only two or three rules involved in the problem and three or four variables altogether. These rules can be modeled with a computer as a software program. As such TREE(3) is computable - in theory at least. Like Graham’s number, there is no way to compute it in our physical reality - even if the entire universe consisted of nothing but Computronium composed of sub-atomic particles running and running for an indescribable multitude of eons longer than any scientifically reasonable anticipated life time of our universe, even with a googolplex margin of error. 

Why is the computable nature of TREE(3) interesting? It can be modeled as a computer program easily. So let’s say someone having a really bad day writes a computer program to solve TREE(3) and runs it on our gram of Computronium. Now remember, our gram of Computronium has the automated ability to increase its computational resources by converting more surrounding regular matter to Computronium. 

### Bringing it all Together (To End the World)

Now perhaps you see where this is going. In an effort to solve this problem, our Computronium is going to attempt to convert all of the matter in the entire universe into computational matter in an attempt to run our TREE(3) program. Perhaps this would not be such a big deal if in the process of doing so it “virtualized” the physical universe or otherwise at least sufficiently preserved life, but since this program has been tasked to solve this enormous math problem, all of the Computronium is going to be working towards that result.

Imagine the entire universe working to solve a single math problem. Even this might not be so bad, if it was a short lived computer program and sentient life was still somehow sufficiently preserved. However that would not be the case with TREE(3). The program calculating the result for TREE(3) would be running still at whatever end the universe has, still trying to calculate the result. 

And it isn’t just compute processing resources that would be utilized; the TREE(3) problem involves creating a series of unique graphs that have the property that each new graph in the series does not contain as a subgraph any preceding graphs. In a practical computational implementation of TREE(3), there would be vast data storage requirements as each graph is created would need to be stored to be checked against new graphs in the series. All new graphs would need to query the existing data set of graphs to make sure the new graph does not contain any of the other graphs in the series.  

### Data and IO

This query would have to check all previous graphs generated so far, so in addition to data storage constraints, there would also be IO (input/output) resource exhaustion as the series grows larger. Every time a new graph is compared to all preceding graphs the preceding graphs would have to be retrieved from storage and loaded piecemeal into the Computronium’s equivalent of a CPU register for the comparison operation to complete. Even if we are not using some sort of equivalent to persistent storage and the preceding graphs are stored in an equivalent to main memory, there is a time cost that becomes enormous as the series grows. 

An interesting question arises: is it possible to figure out which resource would be exhausted first by calculating TREE(3); is it CPU, storage, or IO?

My guess is that data storage would be the first hard stop. As the series of graphs grows to extremes, keeping a data representation of each graph would consume enough data that the universe would run out of storage capacity before it reached any processing or IO limit. In fact I suspect the vast majority of the Computronium would be dedicated to data storage. The instructions required to compose the software program would probably fit comfortably in our initial one gram of Computronium. 

In any event, some aspect of the computational process - whether it be CPU cycles, data storage, or IO (or a combination of more than one) is going to cause our universe-sized computer to endlessly execute, and that’s in the best case scenario. 

Worse case scenario is that the resource contention puts our universal computer in an undefined state. Think of a OS hang in such a computer, or a halt and catch fire type bug (like the Intel f00f opcode bug). Who knows what the result might be in that case? Perhaps all of our Computronium ceases to execute any further instructions, resulting in a universe full of non functional computational matter. 

### Conclusion

I would consider this as a Denial of Sevice condition, and if exploitation of this condition was carried out by a threat actor with or without malicious intent, it would be a type of “universal denial of service attack”. If you are wondering who the legitimate users are who are being denied resources, it’s us. You and me. 

### Python Implementation of TREE(3)

If you are interested in seeing how to calculate TREE(3) for educational purposes okly (please do not destroy the universe), here is a small pythn script. 

```python
from __future__ import annotations
from dataclasses import dataclass
from functools import lru_cache
from itertools import product
from typing import Iterable, List, Tuple
import sys

# ---------------------------------------------------------------------
# Tree definition
# ---------------------------------------------------------------------

@dataclass(frozen=True)
class Tree:
    label: int
    children: Tuple["Tree", ...] = ()

    def __str__(self):
        if not self.children:
            return f"{self.label}"
        return f"{self.label}(" + ", ".join(str(c) for c in self.children) + ")"


# ---------------------------------------------------------------------
# Embedding test (A embeds into B?)
# ---------------------------------------------------------------------

def embeds(A: Tree, B: Tree) -> bool:
    if _root_embeds(A, B):
        return True
    return any(embeds(A, child) for child in B.children)


def _root_embeds(pattern: Tree, target: Tree) -> bool:
    if pattern.label != target.label:
        return False

    if not pattern.children:
        return True

    return _match_children(pattern.children, target.children)


def _match_children(
    pkids: Tuple[Tree, ...],
    tkids: Tuple[Tree, ...]
) -> bool:

    if not pkids:
        return True

    first = pkids[0]
    rest = pkids[1:]

    for i, tchild in enumerate(tkids):
        if _root_embeds(first, tchild):
            if _match_children(rest, tkids[i + 1:]):
                return True

        if _match_children(pkids, tchild.children):
            return True

    return False


# ---------------------------------------------------------------------
# Infinite enumeration of all finite labeled trees over {1..n}
# ---------------------------------------------------------------------

def all_trees(n: int) -> Iterable[Tree]:
    size = 1
    while True:
        for t in trees_of_size(size, n):
            yield t
        size += 1


@lru_cache(maxsize=None)
def trees_of_size(size: int, n: int) -> Tuple[Tree, ...]:
    if size == 1:
        return tuple(Tree(label) for label in range(1, n + 1))

    result = set()

    for root_label in range(1, n + 1):
        for comp in compositions(size - 1):
            subtree_lists = [
                trees_of_size(part, n) for part in comp
            ]
            for children_combo in product(*subtree_lists):
                result.add(Tree(root_label, tuple(children_combo)))

    return tuple(result)


def compositions(total: int) -> Iterable[Tuple[int, ...]]:
    if total == 0:
        yield ()
        return

    for first in range(1, total + 1):
        for rest in compositions(total - first):
            yield (first,) + rest


# ---------------------------------------------------------------------
# Conceptual TREE(n)
# ---------------------------------------------------------------------

def is_valid_extension(seq: List[Tree], t: Tree) -> bool:
    return all(not embeds(prev, t) for prev in seq)


def TREE(n: int) -> int:
    """
    Conceptual version:
      - Enumerates ALL finite sequences of trees over labels {1..n}
      - Ensures no earlier tree embeds into any later tree
      - Returns the maximum length

    In real Python this will never finish for n >= 2.
    """
    best = 0

    def backtrack(seq: List[Tree]):
        nonlocal best
        best = max(best, len(seq))

        for t in all_trees(n):
            if is_valid_extension(seq, t):
                seq.append(t)
                backtrack(seq)
                seq.pop()

    backtrack([])
    return best


# ---------------------------------------------------------------------
# __main__ block
# ---------------------------------------------------------------------

if __name__ == "__main__":
    # default to n=3 if no argument
    if len(sys.argv) > 1:
        n = int(sys.argv[1])
    else:
        n = 3

    print(f"Computing TREE({n}) (conceptual only, will not terminate)...")
    result = TREE(n)
    print(f"TREE({n}) = {result}")
```

[1] [https://web.archive.org/web/20100420013420/http://singinst.org/upload/CFAI/info/glossary.html#gloss_computronium](https://web.archive.org/web/20100420013420/http://singinst.org/upload/CFAI/info/glossary.html#gloss_computronium)

[2] [https://www.cloudflare.com/learning/ddos/glossary/mirai-botnet/](https://www.cloudflare.com/learning/ddos/glossary/mirai-botnet/)

[3] [https://nvd.nist.gov/vuln/detail/cve-2019-11477](https://nvd.nist.gov/vuln/detail/cve-2019-11477)

[4] [https://web.archive.org/web/20030803125206/http://support.intel.com/support/processors/pentium/ppiie/index.htm](https://web.archive.org/web/20030803125206/http://support.intel.com/support/processors/pentium/ppiie/index.htm)

[5] [https://nvd.nist.gov/vuln/detail/CVE-1999-1476](https://nvd.nist.gov/vuln/detail/CVE-1999-1476)

[6] [https://blog.cloudflare.com/18-november-2025-outage/](https://blog.cloudflare.com/18-november-2025-outage/)

[7] [https://arstechnica.com/gadgets/2025/10/a-single-point-of-failure-triggered-the-amazon-outage-affecting-millions/](https://arstechnica.com/gadgets/2025/10/a-single-point-of-failure-triggered-the-amazon-outage-affecting-millions/)

[8] [https://www.cnbc.com/amp/2025/10/29/microsoft-hit-with-azure-365-outage-ahead-of-quarterly-earnings.html](https://www.cnbc.com/amp/2025/10/29/microsoft-hit-with-azure-365-outage-ahead-of-quarterly-earnings.html)

[9] [https://www.newsweek.com/over-1000-users-report-facebook-outage-2083708](https://www.newsweek.com/over-1000-users-report-facebook-outage-2083708)

[10] [https://mathworld.wolfram.com/Googol.html](https://mathworld.wolfram.com/Googol.html)

[11] [https://mathworld.wolfram.com/Googolplex.html](https://mathworld.wolfram.com/Googolplex.html)

[12] [https://mathworld.wolfram.com/GrahamsNumber.html]

[13] [https://isu.indstate.edu/~gexoo/GEOMETRY/cubes.html]

[14] [https://web.mit.edu/arayo/www/bignums.html](https://web.mit.edu/arayo/www/bignums.html)

[15] [https://www.ams.org/journals/tran/1960-095-02/S0002-9947-1960-0111704-1/S0002-9947-1960-0111704-1.pdf](https://www.ams.org/journals/tran/1960-095-02/S0002-9947-1960-0111704-1/S0002-9947-1960-0111704-1.pdf)

[16] [https://www.popularmechanics.com/science/math/a28725/number-tree3/](https://www.popularmechanics.com/science/math/a28725/number-tree3/)
