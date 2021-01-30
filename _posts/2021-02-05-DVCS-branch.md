---
title: Of source control and databases - Part 4 - Branches support
permalink: /dvcs-branch/
tags: [Source Control, Database]
categories: [Of source control and databases]
excerpt: The more the merrier.
---

---

All the code for the articles in this series can be found [here](https://github.com/faouellet/DVCSUS).

---

For this article and the next, we'll be looking at some functionalities that weren't included in my original programming assignment. Up first is the adding branch support to DVCSUS.

## Interface

Even if branches are a supplementary feature, their interface will be like all others: as simple as can be. To this end, I propose to have not one but two commands to deal with branches: ```branch_create``` and ```branch_checkout```. Their purposes are self-explanatory: ```branch_create``` will create a new branch if it doesn't already exists and ```branch_checkout``` will position the repository on a given branch if it exists. It goes without saying that both of these commands will need to receive a branch name as an argument to work. In more concrete terms, we have to implement the following commands:

```bash
dvcsus branch_create <branchname>
dvcsus branch_checkout <branchname>
```

## Repository

### Schema (again)

{%
  include figure.html
  src="/assets/img/repobranches.jpg"
  caption=""
%}

To support branches we need to augment the ```Repository``` database with two new tables: ```Branches``` and ```BranchesCommits``` tables.

* ```Branches```: This will contain all the branches' information. In our prototype system, this'll be limited to a branch's name and its latest commit as identified by a hash.
* ```BranchesCommits```: A link table that keeps track of which commit goes into which branch.

### Metadata

As was the case for the current commit, we'll also need to track the current branch we're working on. To this end, we'll add another entry in the ```Metadata``` table: *CurrentBranch* to which we'll associate the name of the currently active branch.

## Implementation

Evidently, adding branches to DVCSUS will have impacts on what was previously implemented.

* ```Init``` will have to set up the new tables. This means it'll have the reponsibility of creating the *default* branch and setting it as such in the ```Metadata``` table.
* ```Commit``` will also need to consider the current branch when executing. Concretly, this will take the form of adding a new entry in ```BranchesCommits``` and updating ```Branches.HeadCommit``` accordingly.
* ```Add``` won't be affected by the newly present branches (as it should since it only deals in local information).

I won't go into further details about these small refactorings here since they're done at the SQL level. I'd rather we look at C++ source code in this article. However, you're more than welcome to look at the source code if you're curious about what the SQL looks like.

### Creation

Creating a new branch will roughly look like this:

```cpp
[[nodiscard]] bool CreateBranch(const std::string &branchName) noexcept
{
  try
  {
    TDatabasePtr pDB{nullptr, sqlite3_close};
    RETURN_IF(!OpenDatabaseConnection(fs::current_path() / REPO_DB_PATH, pDB), false);

    if (!ValidateNoResult(pDB, fmt::format("SELECT COUNT(*) FROM Branches WHERE Name = \"{}\"", branchName)))
    {
      fmt::print(std::cerr, 
                 fmt::format("Branch '{}' already exists.\n", branchName));
      return false;
    }

    if (ValidateNoResult(pDB, "SELECT COUNT(*) FROM Commits"))
    {
      fmt::print(std::cerr, 
                 fmt::format("Can't create branch '{}' in empty repository.\n", branchName));
      return false;
    }

    const auto createQuery = /* Omitted for brevity */
    return ExecuteQuery(pDB, createQuery);
  }
  catch (const std::exception &e)
  {
      fmt::print(std::cerr, "{}\n", e.what());
      return false;
  }
}
```

Once we get a connection to the ```Repository``` database, we have to do three things with it:

1. Make sure the branch doesn't already exists. This is taken care of by passing a query that gets all the branches with the name of the branch we want to create to ```ValidateNoResult```. If all is well, the query won't return any result.
2. Make sure there are already commits in the repository. This is needed to correctly handle the repository's arborescence. While it's a limitation compared to more full-fledged version control systems, it's acceptable in the context of a prototype.
3. Add the branch to the repository. In other words, add a new entry in the ```Branches``` table which will contains the name of the new branch and the current commit (the *CurrentCommit* entry in the ```Staging.Metadata``` table) as the head of the new branch.

We could also add "checkout the new branch" to the list of operations to do as some source control systems do. I chose not to do it, but you're free to change that!

### Checkout

Checking out a branch could be implemented as shown below:

```cpp
[[nodiscard]] bool CheckoutBranch(const std::string &branchName) noexcept
{
  try
  {
    TDatabasePtr pDB{nullptr, sqlite3_close};
    RETURN_IF(!OpenDatabaseConnection(fs::current_path() / REPO_DB_PATH, pDB), false);
    if (ValidateNoResult(pDB, fmt::format("SELECT COUNT(*) FROM Branches WHERE Name = \"{}\"", branchName)))
    {
      fmt::print(std::cerr, 
                 fmt::format("Can't checkout branch '{}'. It doesn't exists.\n", branchName));
      return false;
    }

    RETURN_IF(!OpenDatabaseConnection(fs::current_path() / STAGING_DB_PATH, pDB), false);
    if (!ValidateNoResult(pDB, "SELECT COUNT(*) FROM Objects"))
    {
      fmt::print(std::cerr, 
                 fmt::format("Can't checkout '{}' branch. Uncommitted changes detected.\n", branchName));
      return false;
    }

    const auto checkoutQuery = /* Omitted for brevity */

    return ExecuteQuery(pDB, checkoutQuery);
  }
  catch (const std::exception &e)
  {
    fmt::print(std::cerr, "{}\n", e.what());
    return false;
  }
}
```

Notice that we need to validate two things here:

1. The branch **must exists**. Checking out a non-existing branch doesn't make any sense.
2. There must be **no uncommitted changes** in the repository. This is in line with all the other distributed source control systems I know of. Since all the current uncomitted changes are based on a given commit, we can't keep them if we are to change branches as this will change the current commit. In fact, it's possible that the uncomitted changes won't even make sense with the new current commit!

As to the query that'll effectively change the current branch, it'll be almost trivial. It'll only have to modify two things, both in the ```Metadata``` table:

* The *CurrentBranch* which will now be the branch name received as argument.
* The *CurrentCommit* which will now be the head commit of the new branch as per the entry in the ```Branches``` table.

That's it. We can now move between branches at will!
