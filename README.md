# IntegrateRNtoExistingApp

Intégrer du React Natif dans une application existante 

## Pré-requis

* RN Development environment (Node js, CLI ...)
* Android Development environment (JDK, Android Studio ...) 
* iOS Development environment (Xcode, Cocoapods ...)

## Pour commencer 

### React Natif
Initialiser le projet RN via la commande: 
``react-native init NameProject``

Supprimer tous les fichiers présents dans les sous-dossiers``/ios`` et ``/android``

### iOS
Créer un projet iOS depuis Xcode. Coller l'ensemble des fichiers du projet dans le sous-dossier ``/ios``
Dans ``/ios``, créer un fichier pod avec ``pod init`` et le modifier avec le **Podfile** suivant:
```podfile
require_relative '../node_modules/react-native/scripts/react_native_pods'
require_relative '../node_modules/@react-native-community/cli-platform-ios/native_modules'

platform :ios, '14.0'

target 'iOSProjectName' do
  config = use_native_modules!

  use_react_native!(
    :path => config[:reactNativePath],
    # to enable hermes on iOS, change `false` to `true` and then install pods
    :hermes_enabled => false
  )

  # Enables Flipper.
  #
  # Note that if you have use_frameworks! enabled, Flipper will not work and
  # you should disable the next line.
  use_flipper!()

  post_install do |installer|
    react_native_post_install(installer)
  end
end
```

Après avoir créer le **Podfile**, installer les pod RN avec ``pod install``

## Naviguer du code natif au RN

### iOS
Créer un boutton dans le ``storyboard``.
Lier ce boutton avec le ``ViewController``

```swift 
import UIKit
import React

class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
    }

    @IBAction func BtnGoReactView(_ sender: Any) {
        let rootView = RNViewManager.sharedInstance.viewForModule("App", initialProperties: nil)
        let reactNativeVC = UIViewController()
        reactNativeVC.view = rootView
        reactNativeVC.modalPresentationStyle = .fullScreen
        present(reactNativeVC, animated: true)
    }
    
}
```
Nous verrons ensuite à quoi correspond le ```swift App```
