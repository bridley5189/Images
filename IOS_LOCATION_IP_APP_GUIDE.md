# iOS App Guide: Show Current Location + Public IP Address

This guide gives you a minimal SwiftUI app setup that:

1. Shows your current location on a map.
2. Uses **satellite view** for the map.
3. Fetches and displays your **public IP address**.

## Do you need APIs?

- **Map display + device location**: **No external API key required** if you use Apple frameworks (`MapKit` + `CoreLocation`).
- **Public IP address**: You need a small web endpoint to discover your public IP. A common option is:
  - `https://api.ipify.org?format=json` (no key required for basic use)

If you prefer a paid provider, keep the key out of source code and inject it at runtime.

## API key prompt template

Use this prompt with your API provider (or secret manager) when generating key integration instructions:

```text
I am building an iOS SwiftUI app that shows the user's current location on a MapKit map in satellite mode and displays the public IP address.
Please provide:
1) The exact HTTPS endpoint URL for getting current public IP in JSON.
2) Required headers and API key format.
3) Rate limits and pricing limits.
4) A sample cURL request and response.
5) Swift URLSession example using API key from Info.plist (not hardcoded).
```

## Info.plist keys you must add

Add these keys to request location access:

- `NSLocationWhenInUseUsageDescription` = `We use your location to show where you are on the map.`
- `NSLocationAlwaysAndWhenInUseUsageDescription` (optional if background behavior needed)

## Minimal SwiftUI example

```swift
import SwiftUI
import MapKit
import CoreLocation

@MainActor
final class LocationViewModel: NSObject, ObservableObject, CLLocationManagerDelegate {
    @Published var region = MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 37.3349, longitude: -122.0090),
        span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)
    )
    @Published var userCoordinate: CLLocationCoordinate2D?
    @Published var ipAddress: String = "Loading IP..."

    private let manager = CLLocationManager()

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestLocation() {
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        userCoordinate = location.coordinate
        region = MKCoordinateRegion(
            center: location.coordinate,
            span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
        )
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        print("Location error: \(error.localizedDescription)")
    }

    func fetchPublicIP() async {
        guard let url = URL(string: "https://api.ipify.org?format=json") else { return }

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let decoded = try JSONDecoder().decode(IPResponse.self, from: data)
            ipAddress = decoded.ip
        } catch {
            ipAddress = "Unable to fetch IP"
        }
    }
}

struct IPResponse: Decodable {
    let ip: String
}

struct ContentView: View {
    @StateObject private var vm = LocationViewModel()

    var body: some View {
        VStack(spacing: 12) {
            Map(coordinateRegion: $vm.region)
                .mapStyle(.imagery) // Satellite view
                .frame(height: 350)
                .clipShape(RoundedRectangle(cornerRadius: 12))
                .padding(.horizontal)

            if let c = vm.userCoordinate {
                Text("Lat: \(c.latitude), Lon: \(c.longitude)")
                    .font(.footnote)
            } else {
                Text("Locating...")
                    .font(.footnote)
            }

            Text("Public IP: \(vm.ipAddress)")
                .font(.headline)

            Spacer()
        }
        .padding(.top)
        .task {
            vm.requestLocation()
            await vm.fetchPublicIP()
        }
    }
}
```

## Privacy and security notes

- Keep API keys in `Info.plist` or secure remote config, never hardcoded.
- Add clear App Store privacy wording for location collection.
- Public IP can change based on network (Wi-Fi/cellular/VPN).
