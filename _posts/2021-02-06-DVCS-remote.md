---
title: Of source control and databases - Part 5 - Remote support (ish)
permalink: /dvcs-remote/
tags: [Source Control, Database]
categories: [Of source control and databases]
excerpt: Not quite there yet, but this is a start.
---

---

All the code for the articles in this series can be found [here](https://github.com/faouellet/DVCSUS).

---

As with the previous article, we'll now look at an extension to my original programming assignment. This time we'll teach DVCSUS about remote repositories.

## Interface

One of the cornerstones of distributed source control is the ability to interact with distant repositories. This is crucial because it enables teams to cooperate by having a common remote repository where they can push or pull modifications. Generally speaking, such a repository is hosted on a network or the Internet (common hosting platforms includes [Github](https://github.com/) and [Bitbucket](https://bitbucket.org/)). In the case of DVCSUS, since it's only a prototype, we'll only offer the possiblity to set a remote using a file path. This will bar DVCSUS from interacting with a repository hosted on the Internet but it will still enable us to interact with a remote repository through a network drive.

Moreover, we'll also provide users with command to push and pull information from the remote repository. Again, we want DVCSUS to stay as simple as possible. This is why the ```push``` and ```pull``` commands won't take any parameters. The remote repository they'll interact with will be the one previously set and no other. If no remote is set, then the commands won't do anything.

All in all, we'll offer the following interface:

```bash
dvcsus set_remote <repopath>
dvcsus push
dvcsus pull
```

## Remote Repository As Metadata

As the remote repository is something that can be set locally, it goes without saying that this information needs to go into the ```Staging``` database. As was shown in earlier articles, this database contains a ```Metadata``` table where we can store any details we want about the local state of the repository. Thus, it's the perfect place to store the remote repository path.

```cpp
[[nodiscard]] bool SetRemote(const fs::path &remoteRepoPath) noexcept
{
  try
  {
    const auto setRemoteQuery{fmt::format("..."), // Omitted for brevity
                              fs::absolute(remoteRepoPath).string())};
    return ExecuteQuery(STAGING_DB_PATH, setRemoteQuery);
  }
  catch (const std::exception &e)
  {
    fmt::print(std::cerr, "{}\n", e.what());
    return false;
  }
}
```

Take note that we make sure to store an absolute path in the database. While there's no problem accepting a relative path as an argument, storing it as such might become a problem later on. The fact is that a relative path could become invalid if the user moves the repository on disk. This would then negatively impact the ```push``` and ```pull``` commands.

## Transfering Data

Linking a local repository with a remote one was the easy part. Now comes the les easy part: transfering data to and from the remote repository.

Yet, it's not actually that difficult once we take into account two observations:

1. For our prototype, the transfer operation is symetrical. In other words, the operation stays the same when pulling or pushing; it's only the source and destination that change.
2. DVCSUS doesn't allow rewriting the past. Therefore, we can only insert new data in the ```Repository``` database but never remove it. This effectively means that transfering data can be implemented in terms of [```INSERT OR IGNORE``` statements](https://www.sqlite.org/lang_insert.html). Everything that's in common between two ```Repository``` databases will remain untouched and what's missing from the destination database will be copied from the source database.

```cpp
enum class TransferDirection { ToLocal, ToRemote };

[[nodiscard]] bool Transfer(TransferDirection direction) noexcept
{
  try
  {
    fs::path source;
    fs::path destination;
    switch (direction)
    {
    case TransferDirection::ToLocal:
      source = GetRemote();
      destination = fs::current_path() / dvcs::REPO_DB_PATH;
      break;
    case TransferDirection::ToRemote:
      destination = GetRemote();
      source = fs::current_path() / dvcs::REPO_DB_PATH;
      break;
    default:
      fmt::print(std::cerr, "Unsupported transfer option\n");
      return false;
    }
    RETURN_IF(source.empty(), false);
    RETURN_IF(destination.empty(), false);
  
    const auto query{
        fmt::format("...", // Lots of INSERT OR IGNORE omitted for brevity
                    source.string())};
    return ExecuteQuery(destination, query);
  }
  catch (const std::exception &e)
  {
    fmt::print(std::cerr, "{}\n", e.what());
    return false;
  }
}
```

## Push And Pull

After the transfer mechanism has been put in place, the ```push``` and ```pull``` commands are trivially easy to implement:

```cpp
[[nodiscard]] bool Pull() noexcept { return Transfer(TransferDirection::ToLocal); }
[[nodiscard]] bool Push() noexcept { return Transfer(TransferDirection::ToRemote); }
```

This is a simple as it gets.

## What About Conflicts?

Good question! In the idealized world of our prototype, there's no such thing as conflicts!
