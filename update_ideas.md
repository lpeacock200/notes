(included code is Ruby/Rails-ish pseudocode)

## Relevant Rails Models + Associations
New model:
```
class FeatureRating
  attribute :rating
  attribute :review_text

  belongs_to :activity
  belongs_to :user
  belongs_to :trip_feature

  enum :rating, { not_recommended: 0, ok: 1, recommended: 2 }, default: :not_recommended
  scope :adequate, -> {where('rating > ?', 0)}
end

# Should be unique to a [user, activity, and feature]
add_index :feature_ratings, [:user_id, :activity_id, :trip_feature_id], unique: true
add_index :feature_ratings, :activity_id
```
- Rating values could also be anchored at 0 for OK with negative values for more negative feedback. Aggregate functions run on results would have negative values offsetting positive values which might be better in some cases. The default value should still be the zero rating.

**feature_ratings**
| Column | Type |
| --- | ----------- |
| id | bigint |
| activity_id | bigint |
| user_id | bigint |
| trip_feature_id | bigint |
| rating | smallint |
| review_text | varchar(1000) |
| created_at | timestamp(6) |
| updated_at | timestamp(6) |

(review_text limit is arbitrary, just some bound)

### Other Relevant Models
(note: using HABTM associations for brevity)
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

  # all available activities
  has_many :activities
end

class Trip
  attribute :id
  belongs_to :traveler, class_name: 'User'
  belongs_to :location

  # The list of activities that were included in this traveler's trip
  has_and_belongs_to_many :activities
  has_and_belongs_to_many :trip_features
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
  has_and_belongs_to_many :activities
  has_and_belongs_to_many :trips
end

class ActivityBooking
  belongs_to :activity
  belongs_to :user
  belongs_to :trip
  attribute :booking_date
  ...
end

```

## Scheduled Jobs
(optional, see notes at bottom)

There will be scheduled jobs to update popularity/ranking tables. They will be run during low traffic time. They can also run in batches (for instance, updating 200 Locations per run, etc.). It could also be limited to Locations of certain popularity.

Popular Location Features Job
- Fetch counts of each distinct preferred TripFeatures in all Trips for a Location. Across all Trips associated with a Location, how often did each TripFeature occur.
- Update location_features_rankings entries for each Location with new popularity_rank values based on the sorted results of counts

Popular Location Actvities Job
- Fetch counts of each distinct Activity across Trips for a Location. Across all Trips associated with a Location, how often was each Activity in a Trip.
- Update location_activities_rankings entries for each Location with new popularity_rank values based on the sorted results of counts

### Related tables

**location_features_rankings**
| Column | Type |
| --- | ----------- |
| id | bigint |
| location_id | bigint |
| trip_feature_id | bigint |
| popularity_rank | integer |
| created_at | timestamp(6) |
| updated_at | timestamp(6) |

(Indexed by location_id)

**location_activities_rankings**
| Column | Type |
| --- | ----------- |
| id | bigint |
| location_id | bigint |
| activity_id | bigint |
| popularity_rank | integer |
| created_at | timestamp(6) |
| updated_at | timestamp(6) |

(Indexed by location_id)


## Activity Review Form Updates
Encapsulate the business logic specific for choosing TripFeatures for a Trip and/or Activity in a class that can be used by the existing reviews and the new final activities reviews controller.
```
class TripActivityFeatures
  # hardcoded or configurable business logic limits
  FEATURE_LIMIT = 3
  SPARSENESS_LIMIT = 2
  ACTIVITY_LIMIT = 5

  def initialize(trip); end

  # Return sparse TripFeatures across the given Trip. This is done by querying across
  # join tables trip_trip_features, activities_trips, and activities_trip_features, group_by
  # trip_feature_id having count <= SPARSENESS_LIMIT.
  def sparse_trip_features; end

  # Return sparse TripFeatures for a given Activity within the Trip. This involves
  # querying for all the given Activity's TripFeatures that are included in the set of
  # ids returned by sparse_trip_feature. (This could likely be condensed into a single query with
  # the query in sparse_trip_features using a subquery or similar). To limit the number of TripFeatures
  # returned to FEATURE_LIMIT, join and sort against the location_features_rankings table prioritizing
  # the most popular features for a Location.
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

  # Reuse the query from 'sparse_trip_features' with the only change
  # being a return of TripFeatures with count > SPARSENESS_LIMIT,
  def non_sparse_trip_features; end

  # Return a hash of TripFeature => [activities]
  # It first fetches TripFeatures from non_sparse_trip_features. For each,
  # it queries to get a set of matching Activities from the Trip. To limit the
  # number of Activities returned, join and reverse sort against the location_activities_rankings
  # table. Reverse sort to surface the least popular activties in a location in the interest of
  # gathering more data and surfacing more/newer activities.
  def activities_by_non_sparse_features; end
end
```
Updates to new controller for Additional Info section
```
class TripFinalReviewController
  ...

  private

  # Called for the get request. First get the non-sparse features and matching
  # activities from TripActivityFeatures::activities_by_non_sparse_features. Then
  # loop through all the Activities to create view objects that encapsulate each
  # TripFeature + Activity + FeatureRating.
  # We are doing this on one page, instead of paginating, to keep with the intention of
  # 'one more thing before you go'. Paginating instead would just involve tracking with
  # a params where in the list of non_sparse_trip_features the page is and then offset+limit or
  # just limit in memory/code since the result set is small.
  def initialize_feature_activities; end

  # Called for the post request. Just receive back a list of Ratings data to store
  # as FeatureRating objects.
  def save_feature_ratings; end
end
```
Questions/thoughts
- Can there ever be existing ratings for these features? If the user could rate from somewhere else, we'd want to filter those selections out of this page.
- From the mockup, we'd need a separate model that represents a specific booking of an Activity by the user on a particular trip, which would include the date and other info. Call it ActivityBooking. We can include that object in the appropriate query result and pass to the view. It shouldn't affect the overall flow of ratings / features for this execise. It's possible it adds one level of indirection as a Trip could have Activities :through => ActivityBookings, etc.
    - By linking a TripRating to an Activity, which will be the same across different Trips for the same User, the User in this case can only give it a rating once. For this scope, that seems OK since having contradicting FeatureRatings for the same Activity from the same User in different Trips means we'd more likely be focusing on aspects of the booking, such as guide, time of year, etc.
- Options for how to limit the Activities per feature on the Final Review Page(s):
  - Prioritize activities linked to fewer trips or ActivityBookings. Advantage is allowing new or lesser known activities to get visibility and collecting more data. This is the option chosen.
  - Prioritize activities with the most trips or ActivityBookings. Advantage is focusing on what most people already like. This may result in a more stagnant user experience as the same activities always rise to the top of searches / automated itineraries.
- To decide on which Activities and TripFeatures are surfaced, we join against popularity / rankings tables that are created and maintained separately. For the first iteration of this feature, the queries to get rankings could be done at the time of request and applied as a subquery or applied as an in-memory filter after Activities or TripFeatures are fetched.
  - The rankings tables could be helpful in other queries / features as it's probably a common filter and sorting requirement




## Misc notes
Super messy query for returning sparse_trip_features, this can be more rails-ified
```
subquery = %Q(SELECT activities_trip_features.trip_feature_id FROM  activities_trip_features inner join activities_trips on activities_trips.activity_id = activities_trip_features.activity_id where activities_trips.trip_id in (?) GROUP BY activities_trip_features.trip_feature_id HAVING (COUNT(activities_trip_features.trip_feature_id) <= ?))
TripFeature.where("id in (#{subquery})", 1, 2)
```
And this query joined and sorted by the rankings table
```
SELECT "trip_features".* FROM "trip_features"
LEFT JOIN location_feature_rankings on location_feature_rankings.trip_feature_id=trip_features.id
WHERE (trip_features.id in (
    SELECT activities_trip_features.trip_feature_id
    FROM  activities_trip_features
    INNER JOIN activities_trips ON activities_trips.activity_id = activities_trip_features.activity_id
    WHERE activities_trips.trip_id in (1)
    GROUP BY activities_trip_features.trip_feature_id
    HAVING (COUNT(activities_trip_features.trip_feature_id) <= 2)))
AND location_feature_rankings.location_id=1
ORDER BY location_feature_rankings.popularity_rank DESC;
```

Example query for Activities matching particular Trip and TripFeature
```
Activity.joins(:trip_features).joins(:trips).where(:trip_features =>{:id => [3]}, :trips => {:id => 1})
```

