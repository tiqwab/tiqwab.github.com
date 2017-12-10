---
layout: post
title: Q&A of IntelliJ Plugin Development
tags: "intellij, java, scala"
comments: true
---

IntelliJ IDEA is a nice IDE supporting various programming languages and helps us to write codes, debug, refactor, test, and so on. Although its default features are good enough, we can add new ones by developing plugins.

This post summarizes my exprience to develop a plugin in the Q&A format. Jetbrains provides comprehensive official documents ([Plugin Development Guidelines][1] and [SDK DevGuide][2]), so here I would like to focus on the more particular situations such as 'How to create function like ...'.

The sample code used in the post is [here][13].

---

### What is the first step to develop new plugins?

- Prepare a IntelliJ plugin project
- Follow the instructions [here][4]
- We can use any programming language appearing in the [*Additional Libraries and Frameworks*] in the [*New Project*] wizard.

The structure of a created project should be like:

```
your-plugin
  +- .idea
  |     +- ...
  |
  +- resources
  |    +- META-INF
  |         +- plugin.xml // plugin settings file
  +- src                  // plugin source directory
  |
  +- your-plugin.iml
```

### How to create new menu items?

- Require the two steps
  - Add `<actions>` in `plugin.xml` to specify where you want to put new items
  - Create a class extending `AnAction` which performs what you want
- Follow the instructions [here][5]

### How to create new intentions?

- What is intention?
  - Editor's suggestion of quickfix, refactoring and etc. (the detail is [here][11])
  - Available by pressing `<Alt> + <Enter>`
- Require the two steps
  - Add `<intentionAction>` in `plugin.xml` to register your intention
  - Create a class extending `IntentionAction` which performs what you want

The below is the sample code of adding a new intention, `HelloIntentionAction` which just println 'hello' when executed on the local variable.

`plugin.xml`:

Add `className` of the new intention into `<extentions>`.

```xml
<idea-plugin>
  ...
  <extensions defaultExtensionNs="com.intellij">
    <!-- Add your extensions here -->
    <intentionAction>
      <className>com.tiqwab.intellij.HelloIntentionAction</className>
    </intentionAction>
  </extensions>
  ...
</idea-plugin>
```

`HelloIntentionAction.scala`:

Prepare `HelloIntentionAction` class.

```scala
package com.tiqwab.intellij

import com.intellij.codeInsight.intention.IntentionAction
import com.intellij.openapi.editor.Editor
import com.intellij.openapi.project.Project
import com.intellij.psi.{PsiFile, PsiLocalVariable}

class HelloIntentionAction extends IntentionAction {

  // Intention name shown in the popup
  override def getText: String = "Hello"

  // ???
  override def getFamilyName: String = "Hello"

  // ???
  override def startInWriteAction(): Boolean = true

  // Define when this intention is available
  override def isAvailable(project: Project,
                           editor: Editor,
                           psiFile: PsiFile): Boolean = {
    // Get PsiElement at the current position
    // What is PsiElement: https://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_elements.html
    val caretModel = editor.getCaretModel
    val offset = caretModel.getOffset
    val psiElement = Option(psiFile.findElementAt(offset))

    // Check if the psiElement is local variable
    psiElement.exists(_.getParent.isInstanceOf[PsiLocalVariable])
  }

  // Define what to do when the intention is executed
  override def invoke(project: Project,
                      editor: Editor,
                      psiFile: PsiFile): Unit = println("Hello")

}
```

In the IDE, intentions are managed and handled like:

- `IntentionManagerImpl.myActions` seems to contains all available intentions.
- `ShowIntentionActionHandler.availableFor` seems to check the intention should be listed in the current condition (call `IntentionAction#isAvailable`)

### How to create new popup menu items?

- What is popup menu?
  - Right click menu in the editor
  - You'll see [*Copy Reference*], [*Paste*], and etc.
- Require the two steps
  - Add `<action>` in `plugin.xml` to register your popup item
  - Create a class extending `AnAction` which performs what you want
- Almost same procedure as described in [Creating an Action][17], but there are some differences.

The below is the sample code of adding a new popup menu action, `PopupHelloAction` which just println 'hello'. This action appears in the popup menu only when the current cursor is on local variables.

`plugin.xml`:

```xml
<idea-plugin>
  ...
  <actions>
    <!-- Popup menu settings -->
    <action id="MyAction.PopupHello" class="com.tiqwab.intellij.PopupHelloAction" text="HelloPopup" description="Hello Popup Action">
      <!-- Specify where you want to put item -->
      <add-to-group group-id="EditorPopupMenu" />
    </action>
  </actions>
</idea-plugin>
```

`PopupHelloAction.scala`:

```scala
package com.tiqwab.intellij

import com.intellij.openapi.actionSystem.{
  AnAction,
  AnActionEvent,
  CommonDataKeys,
}
import com.intellij.openapi.editor.Editor
import com.intellij.psi.{PsiFile, PsiLocalVariable}

/**
  * Sample of popup menu actions
  */
class PopupHelloAction extends AnAction {

  // Enable and disable action
  override def update(e: AnActionEvent): Unit = {
    val dataContext = e.getDataContext
    val isOnLocalVariableOpt = for {
      project <- Option(e.getProject)
      editor <- Option(e.getData[Editor](CommonDataKeys.EDITOR))
      psiFile <- Option(e.getData[PsiFile](CommonDataKeys.PSI_FILE))
      caretModel = editor.getCaretModel
      offset = caretModel.getOffset
      psiElement <- Option(psiFile.findElementAt(offset))
    } yield {
      psiElement.getParent.isInstanceOf[PsiLocalVariable]
    }
    isOnLocalVariableOpt match {
      case None =>
        e.getPresentation.setEnabledAndVisible(false)
      case Some(false) =>
        e.getPresentation.setEnabledAndVisible(false)
      case Some(true) =>
        e.getPresentation.setEnabledAndVisible(true)
    }
  }

  // Perform what to do with the action
  override def actionPerformed(anActionEvent: AnActionEvent): Unit =
    println("hello from popup")
}
```

### How to run and debug plugins?

- Just Run or Debug project in the usual way
- Follow the instruction [here][6]

### Is it possible to use existing Actions?

- Yes, according to [this answer][8]

The below successfully calls `RenameElementAction` from the custom action.

```scala
package com.tiqwab.intellij

import com.intellij.openapi.actionSystem.{AnAction,AnActionEvent,CommonDataKeys}
import com.intellij.refactoring.actions.RenameElementAction

class TellElementAction extends AnAction {

  override def actionPerformed(e: AnActionEvent): Unit = {
    val project = e.getProject
    val element = e.getData(CommonDataKeys.PSI_ELEMENT)
    new RenameElementAction().actionPerformed(e)
  }

}
```

### How to build project?

- Just click [*Build*] -> [*Prepare Plugin Module ...*] in the main menu
  - jar or zip file is created at the project root
- Follow the instruction [here][10]

### Is it possible to install plugins from local archives?

- Yes, you can install plugins packaged as jar or zip following the steps:
  - Click [*File*] -> [*Settings*]
  - Click [*Install plugin from disk...*] in Plugins menu
  - Specify the path of your archive
- [Document][12] says plugins are automatiically installed if put in the appropriate directory, but the IDE does not recognize it in my case

### How to publish new plugins?

- Follow the instruction [here][16]

### Is it possible to manage dependencies to other plugins?

- Add `<depends>` in `plugin.xml`
- Configure project settings to add the dependencies
- Configure run/debug settings to run IntelliJ IDEA with the dependent plugins
- [Plugin Dependencies][14]
- [Developing IDEA plugin with dependency on Scala plugin][15]

---

### References

- [Plugiin Development Guidelines - Intellij Help][1]
- [IntelliJ Platform SDK][2]
- [How to Extend Alt+Enter - Shiraji's Blog][3]
- [http://lab.aratana.jp/entry/2015/02/16/184025][7]

[1]: https://www.jetbrains.com/help/idea/plugin-development-guidelines.html
[2]: http://www.jetbrains.org/intellij/sdk/docs/welcome.html
[3]: http://shiraji.github.io/blog/2016/12/17/how-to-extend-alt+enter/
[4]: http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/creating_plugin_project.html
[5]: http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/creating_an_action.html
[6]: http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/running_and_debugging_a_plugin.html
[7]: http://lab.aratana.jp/entry/2015/02/16/184025
[8]: https://intellij-support.jetbrains.com/hc/en-us/community/posts/206113069-How-to-call-exist-action
[9]: https://stackoverflow.com/questions/23339307/intellij-idea-plugin-development-plugin-not-enabled-on-right-click-on-project
[10]: https://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/deploying_plugin.html
[11]: https://www.jetbrains.com/help/idea/intention-actions.html
[12]: https://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/deploying_plugin.html
[13]: https://github.com/tiqwab/example/tree/master/intellij-plugin-get-started-with-scala
[14]: https://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_dependencies.html
[15]: https://confluence.jetbrains.com/display/SCA/Developing+IDEA+plugin+with+dependency+on+Scala+plugin
[16]: https://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/publishing_plugin.html
[17]: https://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/creating_an_action.html
