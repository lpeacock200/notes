(included code is Ruby/Rails-ish pseudocode)

## Relevant Rails Models + Associations
New model:
```
class FeatureRating
  attribute :rating #smallint
  attribute :review_text #text

  belongs_to :activity
  belongs_to :user
  belongs_to :trip_feature
end

# Should be unique to a user, activity, and feature
add_index :feature_ratings, [:user_id, :activity_id, :trip_feature_id], unique: true
```
Existing models
```
class Activity
  attribute :id
  attribute :name
  has_and_belongs_to_many :trip_features
  has_and_belongs_to_many :trips
  belongs_to :location
end

class Location
  # all trips taking place in this location
  has_many :trips

  # all available activites
  has_many :activites
end

class Trip
  attribute :id
  belongs_to :traveler, class_name: 'User'
  belongs_to :location

  # The list of activities that were included in this traveler's trip
  has_and_belongs_to_many :activities
  has_many :trip_preferences
  has_many :trip_features, through: :trip_preferences
  ...
End

class User
  attribute :id
  attribute :name

  has_many :trips
  # a list of TripFeature that the user is interested in having included in their trip (entered during onboarding)
  has_many :requested_trip_features, class_name: 'TripFeature'
  has_many :feature_ratings
End


class TripFeature
  attribute :id
  attribute :name # eg "Hiking", "Wine Tasting", "Monkeys"
  has_and_belongs_to_many :activites
  has_many :trip_preferences
  has_many :trips, through: :trip_preferences
end

class ActivityBooking
  belongs_to :activity
  belongs_to :user
  belongs_to :trip
  attribute :booking_date
  ...
end

```

## Activity Review Form Updates
Encapsulate the business logic specific for choosing TripFeatures for a Trip and/or Activity in a class that can be used by the existing reviews controller.
```
class TripActivityFeatures
  # hardcoded or configurable business logic limits
  FEATURE_LIMIT = 3
  SPARSENESS_LIMIT = 2
  ACTIVITY_LIMIT = 5

  def initialize(trip); end

  # Return sparse TripFeatures across the given Trip. This is done by querying across
  # join tables activites_trips and activites_trip_features, group_by
  # trip_feature_id having count <= SPARSENESS_LIMIT.
  def sparse_trip_features; end

  # Return all distinct TripFeatures across Trips for a Location sorted by
  # occurrence count, as a measure of popularity. This could be cached to reduce
  # queries, or stored daily/weekly in a table.
  def ranked_preferences_per_trip_location; end

  # Return sparse TripFeatures for a given Activity within the Trip. This involves
  # querying for all the given Activity's TripFeatures that are included in the set of
  # ids returned by sparse_trip_feature. This could be condensed into a single query
  # using a subquery or similar. To limit the number of TripFeatures returned, sort based
  # on the rankings from 'ranked_preferences_per_trip_location' and then limit or
  # truncate the list.
  def sparse_trip_features_for_activity(activity); end
end

```

Rough updates to existing controller
```
class TripReviewController
  ...

  private

  # Called from the action for a get request, assuming there is already a Trip and
  # Activity known. Fetch the list of TripFeatures to rate via
  # TripActivityFeatures::sparse_trip_features_for_activity. Then fetch the existing
  # FeatureRatings matching the TripFeatures ids. Wrap the FeatureRatings objects
  # (new and/or existing) in view objects for the form.
  def initialize_features_to_rate; end

  # Called if the request is a post. Just pulls ratings data from post content
  # and create or updates FeatureRatings given current_user, Activity, and TripFeature
  def save_feature_ratings; end
end

```
Questions:
- Is there a situation where it makes sense to show an existing rating, like if a user returns to the same review url? Assuming yes for now.
- What if a user doesn't leave a rating? Should that score as a FeatureRating with a different rating value to indicate 'skipped'?

## Final Review Page Updates
Add the following methods to the previous TripActivityFeatures class
```
class TripActivityFeatures
  ...

  # Return all distinct Activites across Trips for a Location reverse sorted by
  # occurrence count, as a measure of (non) popularity. This could be cached to reduce
  # queries, or stored daily/weekly in a table. By limiting on the least popular activites
  # we gain more inclusive data. (see notes at bottom)
  def ranked_activites_per_trip_location; end

  # Reuse the query from 'sparse_trip_features' with the only change
  # being a return of TripFeatures with count >= 3.
  def non_sparse_trip_features(limit_count = FEATURE_LIMIT); end

  # Return a hash of TripFeature => [Activites]
  # It first fetches TripFeatures from non_sparse_trip_features. For each,
  # it queries to get a set of matching Activites from the Trip. The Activites
  # are sorted and limited based on the ranks from ranked_activites_per_trip_location.
  def activites_by_non_sparse_features; end
end
```
Updates to new controller for Additional Info section
```
class TripFinalReviewController
  ...

  private

  # Called for the get request. First get the non-sparse features and matching
  # activites from TripActivityFeatures::activites_by_non_sparse_features. Then
  # loop through all the Activites to create view objects that encapsulate each
  # TripFeature + Activity + FeatureRating.
  def initialize_feature_activites; end

  # Called for the post request. Just receive back a list of Ratings data to store
  # as FeatureRating objects.
  def save_feature_ratings; end
end
```
Questions/thoughts
- Can there ever be existing ratings for these features? If the user could rate from somewhere else, we'd want to filter those selections out of this page.
- From the mockup, we'd need a separate model that represents a specific booking of an Activity by the user on a particular trip, which would include the date and other info. Call it ActivityBooking. We can include that object in the appropriate query result and pass to the view. It shouldn't affect the overall flow of ratings / features for this execise. It's possible it adds one level of indirection as a Trip could have Activites :through => ActivityBookings, etc.
- This ^ assumes we don't handle different FeatureRatings for the same Activity in a different trip for the same user.
- Options for how to limit the activites per feature on the Final Review Page(s):
  - Prioritize activites linked to fewer trips or ActivityBookings. Advantage is allowing new or lesser known activites to get visibility and collecting more data.
  - Prioritize activites with the most trips or ActivityBookings. Advantage is focusing on what most people already like. This may result in a more stagnant user experience as the same activites always rise to the top of searches / automated itineraries.
