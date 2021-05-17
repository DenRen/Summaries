Program notes
===

---

## ctags

#### Inctall:
#### sudo apt install exuberant-ctags

#### Examples:
1. ctags main.c
2. ctags *
3. ctags -R *
4. ctags -R /usr/include
5. ctags --c++-kinds=+p --c-kinds=+p -R /usr/include

В *vim* наводим на функцию и нажимаем *Ctrl+]*.\
Но для того, чтобы это сработало, нужно чтобы файлы *tags* были подключены к *vim*.\ Для этого в *~/.vimrc* дописываем *set tags=./tags,tags;$HOME*.

---

## 