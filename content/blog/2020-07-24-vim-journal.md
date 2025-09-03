+++
title = "Vim as a journal"
date = "2020-07-26"
slug = "vim-as-a-journal"
+++

There are many schools of thought on which medium to record your journals in (e.g., physical journal, audio recordings, etc). I like using text files for a couple of reasons:
- Search: it's nice to be able to look for certain events.
- Storage: every journal entry is stored on Dropbox.
- Analysis: lets me answer questions like "How many journal entries did I write this year?"

The biggest draw for me is the writing experience. As I'm already fairly experienced with vim text manipulation, I can write my journals swiftly. It ultimately lowers the barrier for me to write consistently.

While Vim excels at the editing experience, it lacks built-in organization features. This is where [vim-wiki](https://github.com/vimwiki/vimwiki) comes in. Here's an overview of the features:

> With VimWiki, you can:
> - Organize notes and ideas
> - Manage to-do lists
> - Write documentation
> - Maintain a diary
> - Export everything to HTML

There are three features I use the most:
- Index file (`<Leader>ww`)
  - The index file for your journal project. You can think of this as the index page for your journal.
- Create a new daily diary file (`<Leader>w<Leader>w`)
  - Creates a file with the name `YYYY-MM-DD.md`
- Load diary (`<Leader>wi`)
  - This page lists all the journal files you've written under the `./diary` directory
  - It automatically parses the `H1` Markdown header as the title in the index page, so I standardize all titles to be in the format like `<Day of the week> MM/DD @ <Main event of the day>`

### Using a template

I use a `template.md` file as a journal template. The following vimscript achieves three tasks:
- Loads the template
- Formats the header with today's date
- Loads a random vocabulary word from [Kindle's vocab builder](https://yatsushi.com/blog/export-kindle-vocab-builder/)

```txt
function! LoadVocab()
  return systemlist('shuf -n 1 ~/Dropbox/wiki/diary/vocab.txt')
endfunction
function! JournalHeader()
  return "# " . strftime('%a') . " " . strftime('%m/%d') . " @"
endfunction
nmap <Space>l :r ~/Dropbox/wiki/diary/template.md<CR>gg"=JournalHeader()<C-M>p2j"=LoadVocab()<C-M>pgg
```

### Writing plugins

Here are other plugins that help me write better in vim (h/t to [@reedes](https://github.com/reedes) on GitHub):
- [vim-pencil](https://github.com/reedes/vim-pencil) - enables paragraph soft wrapping
- [vim-litecorrect](https://github.com/reedes/vim-litecorrect) - fixes simple typos like `teh` to `the`
- [vim-lexical](https://github.com/reedes/vim-lexical) - enables spell checks

Do you also write a lot in vim? Feel free to reach out with your tips & tricks!
