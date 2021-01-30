---
title: Of source control and databases - Annex A - Testing strategies
permalink: /dvcs-tests/
tags: [Source Control, Database]
categories: [Of source control and databases]
excerpt: How to tell if it's working
---

---

All the code for the articles in this series can be found [here](https://github.com/faouellet/DVCSUS).

---

While programming something new is always fun, there comes a time when we have to put our invention to the test. Can the thing we created stand the test of reality?

So far, you, the reader, had to blindly assume that what I was showing you was indeed working. In this article, I will demonstrate that your faith in me wasn't miguided. To do so, I will show you the two main testing strategies I put in place to assess DVCSUS.

## Validating The Repository's State

The most important thing to test is the state of a DVCSUS repository after a given (set of) operation(s) has been run. To make sure the repository is in the correct state, we need to validate two things:

1. The ```Staging``` and ```Repository``` databases are where they're supposed to be inside the ```.dvcs``` folder.
2. The contents of said databases are what we expected them to be after a given operation.

To this end, for the tests that required it, I put in place databases which were the equivalent of a merge of the ```Staging``` and ```Repository``` databases with the appropriate expected contents. Then, when the time comes to test a given operation, I only need to run a query which [attach](https://www.sqlite.org/lang_attach.html) the test repository's databases together and select the differences between this merger's contents and the expected database's contents (this can be done using the SQL EXCEPT clause). If no differences is found, this means that the tested operation is working as intended.

In terms of code, this validation process looks roughly like this:

```cpp
void ValidateRepositoryContents(const fs::path &expectedDatabasePath) noexcept
{
  sqlite3 *pDBHandle;
  BOOST_REQUIRE(sqlite3_open(dvcs::REPO_DB_PATH.c_str(), &pDBHandle) == SQLITE_OK);
  BOOST_REQUIRE(pDBHandle != nullptr);

  try
  {
    const auto contentQuery = /* Omitted for brevity */
    char *pErrMsg = nullptr;
    auto callback = [](void *count, int argc, char **pArgv, char**) {
      for (int iRes = 0; iRes < argc; ++iRes)
      {
        fmt::print(std::cout, "{}\n", pArgv[iRes]);
      }
      int *pCount = reinterpret_cast<int *>(count);
      ++(*pCount);
      return SQLITE_OK;
    };
    int nbResults{};
    if (sqlite3_exec(pDBHandle, contentQuery.c_str(), callback, &nbResults, &pErrMsg) != SQLITE_OK)
    {
      fmt::print(std::cerr, "SQLite error: {}\n", pErrMsg);
    }
    BOOST_CHECK(nbResults == 0);
    BOOST_REQUIRE(sqlite3_close(pDBHandle) == SQLITE_OK);
  }
  catch (const std::exception &e)
  {
    fmt::print(std::cerr, "{}\n", e.what());
    BOOST_REQUIRE(false);
  }
}
```

## Validating Messages

Although of a somehat lesser importance (especially in the context of a prototype), the messages shown to the user must also be tested. Don't forget that in a *Real World*&trade; context, a message shown at a wrong time (or not shown at all) can lead to a bad user experience.

Since all interactions of DVCSUS are done in the console, recording the messages shown to the user boils down to capturing a standard stream. To do this, I put in place the following ```StreamInterceptor``` class.

```cpp
template <typename TStream> requires std::derived_from<TStream, std::ostream> 
class StreamInterceptor
{
public:
  explicit StreamInterceptor(TStream &stream) : m_originalStream{stream} 
  { 
    m_pOldBuffer = m_originalStream.rdbuf(m_contentStream.rdbuf()); 
  }
  ~StreamInterceptor() { m_originalStream.rdbuf(m_pOldBuffer); }
  std::string GetStreamContent() const
  {
    auto str = m_contentStream.str();
    m_contentStream.str(std::string{});
    return str;
  }

private:
  TStream &m_originalStream;
  mutable std::stringstream m_contentStream;
  std::streambuf *m_pOldBuffer;
};
```

The trick here, if it can be called as such, is that ```StreamInterceptor``` will swap ```stream```'s buffer with its own ```m_contentStream```'s buffer. Thus, any writes to ```stream``` will be redirected to ```m_contentStream```. In other words, this will let us intercept anything that gets written to ```std::cout``` and ```std::cerr```.

Moreover, do take note that ```GetStreamContent``` is intended to be used multiple times in a given test. In fact, its recommended usage is after each DVCSUS operation. Indeed, each time it's called, it clears the ```m_contentStream``` to be ready to give a completely new string to the caller. This saves us the hassle of having to parse and make sense of a large multiline string at the very end of a given test.

Another tool which I've used in combination with ```StreamInterceptor``` is the below ```StartsWith``` function.

```cpp
template <typename TStream>
bool StartsWith(const StreamInterceptor<TStream> &interceptor, 
                std::string_view expected)
{
  const auto content = interceptor.GetStreamContent();
  return content.starts_with(expected);
}
```

The purpose of this function is quite simple: make sure that what ```StreamInterceptor``` has captured so far is as expected. The choice to validate on prefixes was made based on the fact that most of DVCSUS' messages are formatted as to show information unique to a user or repository towards the end. This consequently allows the tests to be independent of both user and location on disk. I will note however that if I had to do more extensive validations on more complex messages, I'd probably would've went with an approach based on regexes instead of one based solely on strings such as ```StartsWith```.

## Sample Test

All of the tools shown in this article can be combined to create clear and concise tests such as this:

```cpp
BOOST_FIXTURE_TEST_CASE(InitCommand, TestFolderFixture)
{
  StreamInterceptor coutInterceptor{std::cout};
  
  try
  {
    BOOST_CHECK(dvcs::Init());

    /* Validate the repository's file structure */
    
    BOOST_CHECK(StartsWith(coutInterceptor, "initialized empty repository:"));
  }
  catch (const std::exception &e)
  {
    fmt::print(std::cerr, "{}\n", e.what());
    BOOST_REQUIRE(false);
  }
  ValidateRepositoryContents("InitTest.db");
}
```
