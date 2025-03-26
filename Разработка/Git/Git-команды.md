>git push --force-with-lease
>git submodule update
>git config --global --add safe.directory D:/Intermedia/Intermedia.NET
>git reset --hard 65bb3aea5dc2fe9654cd8e5e20a3775cfb19296c

> [!info] [Как склеивать коммиты](https://htmlacademy.ru/blog/git/how-to-squash-commits-and-why-it-is-needed)
> git cherry -v develop
> git cherry -v develop | wc -l
> git rebase -i HEAD~5

> [!info] [Как откатить оддельные файлы](https://ru.linux-console.net/?p=7833)
> 

> [!info] Ресетнуться на голову develop ветки
> git reset $(git merge-base develop $(git rev-parse --abbrev-ref HEAD))

> [!info] 7-ZIP
> winget install -e --id 7zip.7zip
