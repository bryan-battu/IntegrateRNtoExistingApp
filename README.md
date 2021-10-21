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
Créer un projet iOS depuis Xcode. Coller l'ensemble des fichiers du projet dans le sous-dossier ``/ios``.
Créer ensuite un fichier pod avec ``pod init`` dans le même sous-dossier et le modifier avec le **Podfile** suivant:
```ruby
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

### RN
Créer un fichier ``RNView.js`` à la base du projet.
Ajouter le code suivant 
```js
'use strict';

import React from 'react';
import ReactNative, {
    StyleSheet,
    Text,
    View,
} from 'react-native';

const styles = StyleSheet.create({
    container: {
        flex: 1,
        justifyContent: 'center',
        alignItems: 'center',
        backgroundColor: 'green',
    },
    welcome: {
        fontSize: 20,
        color: 'white',
    },
});

class AppName extends React.Component {
    render() {
        return (
            <View style={styles.container}>
                <Text style={styles.welcome}>We're live from React Native!!!</Text>
            </View>
        )
    }
}

module.exports = AddRatingApp;
```
Ce code va créer une vue simple affichant un écran vert et du texte

### iOS
Dans le projet Xcode, créer un boutton dans le ``storyboard``.
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
        let rootView = RNViewManager.sharedInstance.viewForModule("AppName", initialProperties: nil)
        let reactNativeVC = UIViewController()
        reactNativeVC.view = rootView
        reactNativeVC.modalPresentationStyle = .fullScreen
        present(reactNativeVC, animated: true)
    }
    
}
```
``"AppName"`` correspond au nom du module RN qui va s'afficher.

Apple a bloqué le chargement implicite de ressources HTTP en clair. Nous devons donc ajouter ``NSAppTransportSecurity`` dans le ``Info.plist``

Dans Xcode, ouvrir ``Info.plist``
* Ajouter ``NSAppTransportSecurity`` en tant que Dictionnary.
* Ajouter ``NSExceptionDomains`` en tant que Dictionnary en dessous de ``NSAppTransportSecurity``.
* Ajouter une clé nommée ``localhost`` de type Dictionnary en dessous de ``NSExceptionDomains``.
* Ajouter ``NSTemporaryExceptionAllowsInsecureHTTPLoads`` avec la valeur ``YES`` en dessous de ``localhost``.

![Capture d’écran 2021-10-21 à 14 37 28](https://user-images.githubusercontent.com/57012683/138278649-951f746e-ada2-45d0-9508-a2df9699067f.png)

## Créer un module natif pour vérifier la capacité de communiquer entre RN et code natif

Nous allons créer un TextField ainsi qu'un boutton qui envoie au natif le text écrit dans le TextField.

### iOS
Un module natif iOS a besoin de 2 fichiers:
* Un fichier ``.swift``que nous appellerons ``TestConnectNative.swift``:
```swift 
import Foundation
import React

@objc(TestConnectNativeModule)
class TestConnectNativeModule: NSObject {
    @objc
    static func requiresMainQueueSetup() -> Bool {
        return true
    }
    
    @objc
    func sendMessageToNative(_ rnMessage: String) {
        print("This log is from swift: \(rnMessage)")
    }
}
```
* un fichier ```.m``` pour exposer nos méthodes au RN que nous appellerons ``TestConnectNative.m``:
```objective-c
#import <React/RCTBridgeModule.h>

@interface RCT_EXTERN_REMAP_MODULE(TestConnectNative, TestConnectNativeModule, NSObject)

RCT_EXTERN_METHOD(sendMessageToNative: (NSString)rnMessage)

@end
```

### RN
Importer ``NativeModules, Button``et ``TextInput``.
Au dessus de votre classe ```App Name```, ajouter le code suiant:
```js
const testConnectNative = NativeModules.TestConnectNative;

const TestConnectNative = {
    sendMessage: msg => {
        testConnectNative.sendMessageToNative(msg);
    }
}
```
Dans le ```render``` en dessous du texte, ajouter le code suivant:
```js
<TextInput
    placeholder={'Typing some messages to native...'}
    onChangeText={newText => this.setState({messageToNative: newText})}
/>
<Button
   title="Send message to native"
   color="#841584"
   onPress={() => {TestConnectNative.sendMessage(this.state.messageToNative)}}
/>
```
## Sortir du RN pour revenir au natif
## iOS
Créer un fichier nommé ```RNViewManager.swift```.
```swift 
import Foundation
import React

class RNiOSViewManager: NSObject {
    var bridge: RCTBridge?
    
    static let sharedInstance = RNiOSViewManager()
    
    func createBridgeIfNeeded() -> RCTBridge {
        if bridge == nil {
            bridge = RCTBridge.init(delegate: self, launchOptions: nil)
        }
        return bridge!
    }
    
    func viewForModule(_ moduleName: String, initialProperties: [String: Any]?) -> RCTRootView {
        let viewBridge = createBridgeIfNeeded()
        let rootView: RCTRootView = RCTRootView(
            bridge: viewBridge, moduleName: moduleName, initialProperties: initialProperties
        )
        return rootView
    }
}

extension RNiOSViewManager: RCTBridgeDelegate {
    func sourceURL(for bridge: RCTBridge!) -> URL! {
            return URL(string: "http://localhost:8081/index.bundle?platform=ios")
    }
}
```

### Test
Dans le Terminal, se placer dans le dossier RN et taper la commande ```npm start```
Vous devriez voir quelque chose comme: 
```shell 
> RN@0.0.1 start /Users/.../.../RN
> react-native start

                                                      
                        #######                       
                   ################                   
                #########     #########               
            #########             ##########          
        #########        ######        #########      
       ##########################################     
      #####      #####################       #####    
      #####          ##############          #####    
      #####    ###       ######       ###    #####    
      #####    #######            #######    #####    
      #####    ###########    ###########    #####    
      #####    ##########################    #####    
      #####    ##########################    #####    
      #####      ######################     ######    
       ######        #############        #######     
         #########        ####       #########        
              #########          #########            
                  ######### #########                 
                       #########                      
                                                      
                                                      
                    Welcome to Metro!
              Fast - Scalable - Integrated



To reload the app press "r"
To open developer menu press "d"
```
Lancer ensuite votre application depuis Xcode. 
Essayez d'entrer du texte dans le TextField puis cliquer sur le boutton.
Le message 
```shell
This log is from swift: {Votre texte}
```
devrait apparaitre dans le terminal Xcode.
Si c'est le cas, alors tout fonctionne.

## Passer en mode production
Jusqu'à maintant, nous avions besoin de lancer Metro avec `npm start` afin de faire fonctionner le module en React.
Nous allons faire en sorte que ce ne soit plus le cas.

Pour cela, il y'a plusieures étapes:
* Dans le *Project Navigator*, cliquer sur le projet > Build Phases > cliquer sur + et sélectionner `New Run Script Phase`
* Ajouter le script suivant:
```shell
export NODE_BINARY=node
../node_modules/react-native/scripts/react-native-xcode.sh
```
* Dans `RNViewManager`, modifier la ligne:
```swift 
return URL(string: "http://localhost:8081/index.bundle?platform=ios")
```
par la ligne:
```swift 
return Bundle.main.url(forResource: "main", withExtension: "jsbundle")
```
* Aller au dossier RN dans le terminal et taper la commande:
```shell
RN % react-native bundle --platform ios --dev false --entry-file index.js --bundle-output main.jsbundle --assets-dest {Chemin de destination}
```
Cette commande va créer un fichier ```main.jsbundle```
* Coller ce fichier dans votre projet Xcode:

![image](https://user-images.githubusercontent.com/57012683/138277915-2363ec78-2fe7-4dfd-b1c6-6bff794d59ea.png)
* Dans le menu d'Xcode, aller dans *Product* > *Scheme* > *Edit Scheme* et changer *Build Configuration* de `Debug` à `Release`

Lancer à nouveau l'application en fermant le terminal.
Le module de React apparait.
