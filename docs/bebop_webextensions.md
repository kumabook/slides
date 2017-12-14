## bebop WebExtensions



### What's bebop

- Groovy webtension that offers command line interface like emacs helm for browsing
- Alternative to vimperator
- [source](https://github.com/kumabook/bebop)
- Firefox and chrome
  - [Firefox addon](https://addons.mozilla.org/ja/firefox/addon/bebop/)
  - [Chrome extension](https://chrome.google.com/webstore/detail/bebop/idiejicnogeolaeacihfjleoakggbdid)



### Motivation

- vimperator doesn't work in Firefox 57+
- Other alternatives exist:
  - [tridactyl](https://github.com/cmcaine/tridactyl)
  - [vim-vixen](https://github.com/ueokande/vim-vixen)
    - Shortcut keys work only in content.
- Q. What is it that I want?
  - A. command line tool for browsing
    - My answer "helm on browser"



### bebop

<img src="./images/bebop_popup.png" style="border: none;">


## Specification

1. Show popup cli with `C-Comma` (WebExtensions browser_action)
2. search candidates from several sources.
3. narrow down them with text filtering
4. run a command with selected candidates




## Archtecture

<img src="./images/bebop_arch.png" style="border: none;">



## Sources

search candidates from these sources:

- search
- link
- tab
- history
- bookmark


## Sources: link

- Get links (clickable elements) in the current tab
  - content_script of WebExtensions
  - `a`
  - `button`
  - `input[type="button"]`
  - `input[type="submit"]`
  - `[role="button"]`
- Add link marker and highlight selected element
  - tips: only visible elements



# popup

<img src="./images/bebop_popup.png" style="border: none;">


## popup: filtering

- filtering with type
  - l ... link
  - t ... tab
  - b ... bookmark
  - h ... history
- ex: `l 阿部寛`
  - narrow down to link candidates searched with `阿部寛`


### popup: command select

- list  avaidable commands about selected candidate
- `C-i`
  - show command list for current candidate

<img src="./images/bebop_command_list.png" height="150px" style="border: none;">


### popup: multiple candidates

- mark candidates with `C-SPC`
- run a command with candidates
- Some command supports multiple candidate
  - open url
  - close tab
  - delete bookmark
  - delete history



### Command

- command consists of `label`, `icon`, `handler` and `contentHandler`
- label
  - string that is displayed in popup ui
- icon
  - icon image for popup ui


## Command: handler

- handler (background handler)
  - full access to WebExtensions API
  - `Array<Candidate> -> Promise<Any>`
- content handler
  - restricted access to WebExnteions API
  - access to dom API
  - `Array<Candidate> -> Promise<Any>`


### Development

- popup ... ui
  - react + redux + redux-saga
- content_script
  - run functions by popup message
- background script
  - manage popup and content_script


### Restriction

- `C-n` can't be override
  - security reason?
- window can't focus the popup input immediately



### Future work

- Add more built-in command
  - navigation
  - add bookmark
  - cookie manager
  - history manager
- setting page
  - default candidates order
  - default candidates number


### Future work: plugin

- sources
  - web based
  - message passing between anoter WebExntenions
- commands
  - register a command dynamically for cetern candidate type
