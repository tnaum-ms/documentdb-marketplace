# DocumentDB Query Insights (prototype)

A canvas for the GitHub Copilot app that helps you understand and speed up DocumentDB queries
from their explain output. It works with DocumentDB and other MongoDB API explain plans.

This is an experimental prototype. It is not supported and may change or be removed.

## What it does

- Paste the output of `.explain("executionStats")` into chat, or into the panel's paste box.
- The side panel shows a performance rating, key metrics, the stages the query ran, and where
  the time went.
- **Explain this stage**, **Explain this plan to me** and **How can I make this faster?** ask
  Copilot in chat. Copilot may ask for index definitions or collection stats to sharpen the
  answer; share them in chat.

## Your data

- The canvas never connects to a database and holds no credentials.
- When you press a button, the explain output you pasted and a description of what the panel
  shows are attached to your chat request and sent to the model. Explain output can contain
  query values; remove sensitive values before pasting.

## More

Do more in VS Code with the
[DocumentDB extension for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-documentdb).

Licensed under the MIT License; see `LICENSE`.
