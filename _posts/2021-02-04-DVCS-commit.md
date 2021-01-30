---
title: Of source control and databases - Part 3 - Committing changes
permalink: /dvcs-commit/
tags: [Source Control, Database]
categories: [Of source control and databases]
excerpt: Look at my changes!
---

---

All the code for the articles in this series can be found [here](https://github.com/faouellet/DVCSUS).

---

In this article, we'll explain how we can make public previously staged modifications with the ```commit``` command.

## Interface

To continue with the theme of simplicity, the ```commit``` command will also be as bare bones as can be. This means that we'll restrict its parameter list to only three:

```bash
dvcsus commit <author> <email> <message>
```

* \<author> is the name of the author of the commit
* \<email> is the email of the author of the commit
* \<message> is a message describing the commit.

One could argue that we could eschew the \<author> and \<email> parameters and instead store these information in the ```Staging``` database. More precisely, these information could be stored in the ```Metadata``` table. This would indeed be an appropriate use of the ```Metadata``` table. The reason that I didn't go that way is that I wanted to implement the same exact interface I asked my students to implement in my original programming assignment.

## Repository

### Schema

As was previously explained, a DVCSUS repository is made of two databases. The ```Staging``` database that is used to store local information was shown in the previous article. For the ```commit``` command, we'll work with the second database: ```Repository```. First, let's have a look at its schema:

{%
  include figure.html
  src="/assets/img/repo.jpg"
  caption=""
%}

These tables have the following meanings:

* ```Objects```: A replica of the ```Objects``` table in the ```Staging``` database as far as the schema goes. A big component of the commit process is transfering the objects in the ```Staging``` database to the ```Repository``` database. This is what makes the modifications to the repository public since the ```Repository``` database is public (understand shared here) between all who have access to a repository.
* ```Commmits```: A description of a set of modifications to the repository. As with a regular source control system, every commit in DVCSUS is identified by a SHA1 hash of its contents (author's name, author's email, commit message and its parent's hash). Yes, we could've make do with a simple row ID to identify a commit and represent an arborescence. Still, a hash is another tool with which we can validate the content of the repository. This is another good reason to use it in our system.
* ```CommitsObjects```: A simple link table. It's necessary to keep track of which objects appears in a given commit.

### Metadata

While the goal of the ```commit``` command is to make modifications global to the repository, it'll still touch on a bit of local information. More precisely, we'll need to keep track of the current head commit (the last commit that was created in the ```Repository``` database). This is so we can create new commits (each commit except the root commit must refer to a parent commit) as well as position ourselves on the commit that we want (not implemented in this prototype). Since the ```Metadata``` table is uber simple, storing this information merely consists in associating a name (say *CurrentCommit*) with a hash value (the last created commit's hash).

### Transactions

As you can probably guess with what you've read until now, the ```commit``` command needs to operate on multiple databases at once. Furthermore, it needs to operate on these databases **transactionally**. For instance, if, for some reason, we can't successfully create a commit, not only the ```Repository``` database musn't change, the ```Staging``` database must also stay the same (not setting a new *CurrentCommit*). Thankfully, there are already DBMS that offer features which can fulfill this requirement. In DVCSUS' case, the choice fell on [SQLite](https://www.sqlite.org/index.html).

### Implementation

In the end, the ```commit``` command has a really straightforward implementation. The ```Repository.Objects``` table is filled with what's currently in the ```Staging.Objects``` table. The latter is then cleared. Afterward, a commit is inserted in the ```Repository.Commits``` table with the arguments received plus a hash created with the already in place ```CommmitSHA1``` function. The same hash is also set as the new *CurrentCommit* in the ```Staging.Metadata``` table. There's only the matter of the parent's hash which has to be fetched from the ```Metadata``` table which add some complexity to the implementation. Without it, everything could be done inside a single SQL script.

```cpp
[[nodiscard]] bool Commit(const std::string &author, 
                          const std::string &email, 
                          const std::string &message) noexcept
{
  for (const auto &arg : {author, email, message})
  {
    if (arg.empty())
    {
      fmt::print(std::cerr, "Can't commit. Missing information\n");
      return false;
    }
  }
    
  std::string hash;
  auto callback = /*Assign hash with the query result*/
  RETURN_IF(!ExecuteQuery(STAGING_DB_PATH, 
                          "SELECT Value FROM Metadata WHERE Name = \"CurrentCommit\";", 
                          callback, 
                          &hash), 
            false);

  std::vector<char> commitData;
  commitData.insert(commitData.end(), author.cbegin(), author.cend());
  commitData.insert(commitData.end(), email.cbegin(), email.cend());
  commitData.insert(commitData.end(), message.cbegin(), message.cend());
  commitData.insert(commitData.end(), hash.cbegin(), hash.cend());
  const auto commitHash = ComputeSHA1(commitData);

  try
  {
      const auto stagingFullPath = fs::current_path() / STAGING_DB_PATH;
      const auto commitQuery = fmt::format("...", // Omitted for brevity
          stagingFullPath.string(), commitHash, author, email, message);

      RETURN_IF(!ExecuteQuery(REPO_DB_PATH, commitQuery), false);
  }
  catch (const std::exception &e)
  {
      fmt::print(std::cerr, "{}\n", e.what());
      return false;
  }

  return true;
}
```

Up next, we'll go into some extensions to the original programming assignment that inspired this series.