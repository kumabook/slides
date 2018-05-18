## bebop WebExtensions



### About

- Motivation: I like emacs and emacs helm.
  - Use browser like cli
- Firefox and chrome (and vivaldi)
  - [Firefox addon](https://addons.mozilla.org/ja/firefox/addon/bebop/)
  - [Chrome extension](https://chrome.google.com/webstore/detail/bebop/idiejicnogeolaeacihfjleoakggbdid)




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
- session
- command
- hatebu



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

- open url
- delete bookmark
- manage cookie
- ...




### Development

- popup ... ui
  - react + redux + redux-saga
- content_script
  - highlight links
- background script
  - manage popup and content_script
  - run command with webextension api
  - save hatena bookmarks to indexedDB
- options_ui
  - save options with storage api



### Restriction

- `C-n` can't be override
  - security reason?
- windows can't focus the popup input immediately
